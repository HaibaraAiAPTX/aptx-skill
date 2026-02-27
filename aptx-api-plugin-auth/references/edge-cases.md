# 边缘情况处理

本文档包含认证插件使用中可能遇到的边缘情况处理模式。

## SSR 会话隔离

SSR 环境下，不同用户的请求必须使用独立的 store 实例，避免 token 串用：

### Next.js App Router

```ts
import { cookies } from "next/headers";
import { createAuthMiddleware } from "@aptx/api-plugin-auth";
import { createSsrCookieTokenStore } from "@aptx/token-store-ssr-cookie";

// ✅ 正确：使用异步工厂函数，每请求独立 store
const auth = createAuthMiddleware({
  store: async () => {
    const cookieStore = await cookies();
    return createSsrCookieTokenStore({
      tokenKey: "aptx_token",
      getCookieHeader: () => cookieStore.toString(),
      setCookie: (v) => parseAndSetCookie(cookieStore, v),
    });
  },
  refreshToken: async () => { /* ... */ },
});

// ❌ 错误：共享 store 实例会导致 token 串用
const sharedStore = new SomeStore();
const auth = createAuthMiddleware({
  store: sharedStore, // 危险！所有请求共享同一个 store
});
```

### Next.js Pages Router

```ts
import { createAuthMiddleware } from "@aptx/api-plugin-auth";
import { createSsrCookieTokenStore } from "@aptx/token-store-ssr-cookie";

export function getServerSideProps(ctx) {
  // 使用同步工厂函数，从请求上下文创建 store
  const auth = createAuthMiddleware({
    store: () => createSsrCookieTokenStore({
      getCookieHeader: () => ctx.req.headers.cookie ?? "",
      setCookie: (v) => ctx.res.setHeader("Set-Cookie", v),
    }),
    refreshToken: async () => { /* ... */ },
  });

  // ...
}
```

### 并发请求隔离

异步工厂函数确保并发请求各自独立：

```ts
// 用户 A 和用户 B 同时发起请求
// 每个请求都会调用工厂函数，创建独立的 store 实例
// 请求 A 的 store 只能读取用户 A 的 cookie
// 请求 B 的 store 只能读取用户 B 的 cookie
```

## 并发刷新防止

中间件内部已实现防并发刷新机制，多个同时请求 401 时只会触发一次刷新：

```ts
// 场景：3 个请求同时返回 401
// 结果：只调用一次 refreshToken，所有请求等待刷新完成后重试
```

## Token 验证

在 `refreshToken` 中添加验证逻辑：

```ts
refreshToken: async () => {
  const res = await fetch("/api/refresh");
  const data = await res.json();

  // 验证返回数据格式
  if (!data.token) {
    throw new Error("Invalid token response: missing token");
  }

  return { token: data.token, expiresAt: data.expiresAt };
}
```

## 网络错误重试

在 `refreshToken` 中处理临时网络错误：

```ts
refreshToken: async () => {
  const maxRetries = 3;
  for (let i = 0; i < maxRetries; i++) {
    try {
      const res = await fetch("/api/refresh");
      if (!res.ok) throw new Error(`Refresh failed: ${res.status}`);
      return await res.json();
    } catch (error) {
      if (i === maxRetries - 1) throw error; // 最后一次重试失败则抛出
      await new Promise(r => setTimeout(r, 1000 * (i + 1))); // 指数退避
    }
  }
}
```

## 跨标签页 Token 同步

当用户打开多个标签页时，需要同步 token 状态。推荐使用 cookie 存储（`@aptx/token-store-cookie`）：

```ts
import { createCookieTokenStore } from "@aptx/token-store-cookie";

// Cookie 自动在标签页间共享
const store = createCookieTokenStore({
  tokenKey: "token",
  metaKey: "token_meta",
  syncExpiryFromMeta: true,
});
```

如果使用自定义的 localStorage 实现，需要监听 storage 事件：

```ts
window.addEventListener("storage", (e) => {
  if (e.key === "token") {
    // Token 在其他标签页更新，重新加载当前页面
    window.location.reload();
  }
});
```

## 无限刷新保护

防止刷新失败导致的无限循环：

```ts
const auth = createAuthMiddleware({
  store,
  refreshToken: async () => {
    const res = await fetch("/api/refresh");
    if (!res.ok) throw new Error("Refresh failed");
    return await res.json();
  },
  shouldRefresh: (error, req, ctx) => {
    // 避免在刷新请求本身失败时重试
    if (req.url?.includes("/api/refresh")) {
      return false;
    }
    return error.status === 401;
  },
});
```
