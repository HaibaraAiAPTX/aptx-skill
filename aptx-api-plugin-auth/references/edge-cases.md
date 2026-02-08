# 边缘情况处理

本文档包含认证插件使用中可能遇到的边缘情况处理模式。

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
