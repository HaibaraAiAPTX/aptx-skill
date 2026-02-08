# Testing Guide

## 测试工具

使用 vitest 进行测试：

```ts
import { describe, expect, it } from "vitest";
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";
import { Context } from "@aptx/api-core";
import { Request } from "@aptx/api-core";
import { Response } from "@aptx/api-core";
```

## 基础测试：添加 header

测试中间件从 cookie 读取并添加 header：

```ts
describe("CSRF Middleware", () => {
  it("从 cookie getter 添加 header", async () => {
    const mw = createCsrfMiddleware({
      getCookie: (name) => (name === "XSRF-TOKEN" ? "test-token-123" : undefined),
    });

    const res = await mw.handle(
      new Request({ method: "POST", url: "https://example.com/api" }),
      new Context({ id: "test-1", signal: new AbortController().signal }),
      async (req) => {
        expect(req.headers.get("X-XSRF-TOKEN")).toBe("test-token-123");
        return new Response({
          status: 200,
          headers: new Headers(),
          url: "https://example.com/api",
          raw: {},
        });
      }
    );

    expect(res.status).toBe(200);
  });
});
```

## 测试：同源策略

测试 `sameOriginOnly` 配置：

```ts
it("sameOriginOnly=true 时跳过跨域请求的 token", async () => {
  const mw = createCsrfMiddleware({
    sameOriginOnly: true,
    getCookie: () => "test-token-123",
  });

  // Mock window.location 用于测试
  const originalWindow = (globalThis as any).window;
  (globalThis as any).window = {
    location: { origin: "https://example.com", href: "https://example.com" },
  };

  try {
    const res = await mw.handle(
      new Request({ method: "POST", url: "https://api.example.com/data" }),
      new Context({ id: "test-2", signal: new AbortController().signal }),
      async (req) => {
        expect(req.headers.get("X-XSRF-TOKEN")).toBeNull();
        return new Response({
          status: 200,
          headers: new Headers(),
          url: "https://api.example.com/data",
          raw: {},
        });
      }
    );

    expect(res.status).toBe(200);
  } finally {
    (globalThis as any).window = originalWindow; // 始终恢复
  }
});
```

## 测试：缺失 cookie

测试 cookie 不存在时的行为：

```ts
it("优雅处理缺失的 cookie", async () => {
  const mw = createCsrfMiddleware({
    getCookie: () => undefined, // 未找到 cookie
  });

  await mw.handle(
    new Request({ method: "POST", url: "https://example.com/api" }),
    new Context({ id: "test-3", signal: new AbortController().signal }),
    async (req) => {
      expect(req.headers.get("X-XSRF-TOKEN")).toBeNull();
      return new Response({
        status: 200,
        headers: new Headers(),
        url: "https://example.com/api",
        raw: {},
      });
    }
  );
});
```

## 测试：自定义配置

测试自定义 cookie/header 名称：

```ts
it("使用自定义 cookie/header 名称", async () => {
  const mw = createCsrfMiddleware({
    cookieName: "MY-CSRF-TOKEN",
    headerName: "X-MY-CSRF",
    getCookie: (name) => (name === "MY-CSRF-TOKEN" ? "custom-token" : undefined),
  });

  await mw.handle(
    new Request({ method: "POST", url: "https://example.com/api" }),
    new Context({ id: "test-4", signal: new AbortController().signal }),
    async (req) => {
      expect(req.headers.get("X-MY-CSRF")).toBe("custom-token");
      return new Response({
        status: 200,
        headers: new Headers(),
        url: "https://example.com/api",
        raw: {},
      });
    }
  );
});
```

## 测试：URL 解码

测试 cookie 值的 URL 解码：

```ts
it("自动 URL 解码 cookie 值", async () => {
  const mw = createCsrfMiddleware({
    getCookie: (name) => (name === "XSRF-TOKEN" ? "test%20value%26symbol" : undefined),
  });

  await mw.handle(
    new Request({ method: "POST", url: "https://example.com/api" }),
    new Context({ id: "test-5", signal: new AbortController().signal }),
    async (req) => {
      // 应该是解码后的值
      expect(req.headers.get("X-XSRF-TOKEN")).toBe("test value&symbol");
      return new Response({
        status: 200,
        headers: new Headers(),
        url: "https://example.com/api",
        raw: {},
      });
    }
  );
});
```

## 测试：空 cookie

测试空字符串 cookie：

```ts
it("空字符串 cookie 处理", async () => {
  const mw = createCsrfMiddleware({
    getCookie: (name) => (name === "XSRF-TOKEN" ? "" : undefined),
  });

  await mw.handle(
    new Request({ method: "POST", url: "https://example.com/api" }),
    new Context({ id: "test-6", signal: new AbortController().signal }),
    async (req) => {
      // 应该添加空字符串 header
      expect(req.headers.get("X-XSRF-TOKEN")).toBe("");
      return new Response({
        status: 200,
        headers: new Headers(),
        url: "https://example.com/api",
        raw: {},
      });
    }
  );
});
```

## 测试：sameOriginOnly=false

测试所有请求都添加 token：

```ts
it("sameOriginOnly=false 时所有请求都添加 token", async () => {
  const mw = createCsrfMiddleware({
    sameOriginOnly: false,
    getCookie: () => "test-token",
  });

  await mw.handle(
    new Request({ method: "POST", url: "https://api.example.com/data" }),
    new Context({ id: "test-7", signal: new AbortController().signal }),
    async (req) => {
      expect(req.headers.get("X-XSRF-TOKEN")).toBe("test-token");
      return new Response({
        status: 200,
        headers: new Headers(),
        url: "https://api.example.com/data",
        raw: {},
      });
    }
  );
});
```

## 测试：SSR 环境

测试 SSR 环境（无 window）：

```ts
it("SSR 环境默认 isSameOrigin 返回 true", async () => {
  const originalWindow = (globalThis as any).window;
  (globalThis as any).window = undefined; // 模拟 SSR

  try {
    const mw = createCsrfMiddleware({
      sameOriginOnly: true,
      getCookie: () => "ssr-token",
    });

    await mw.handle(
      new Request({ method: "POST", url: "https://api.example.com/data" }),
      new Context({ id: "test-8", signal: new AbortController().signal }),
      async (req) => {
        // SSR 环境默认视为同源
        expect(req.headers.get("X-XSRF-TOKEN")).toBe("ssr-token");
        return new Response({
          status: 200,
          headers: new Headers(),
          url: "https://api.example.com/data",
          raw: {},
        });
      }
    );
  } finally {
    (globalThis as any).window = originalWindow;
  }
});
```

## 测试最佳实践

### 始终恢复 mock

```ts
const originalWindow = (globalThis as any).window;
try {
  // 测试代码
} finally {
  (globalThis as any).window = originalWindow;  // 必须在 finally 中恢复
}
```

### 覆盖所有场景

- ✅ 添加 header 的基本场景
- ✅ 同源策略（同源/跨域）
- ✅ 缺失 cookie
- ✅ 自定义配置
- ✅ URL 解码
- ✅ 空值处理
- ✅ SSR 环境
- ✅ 无效 URL
