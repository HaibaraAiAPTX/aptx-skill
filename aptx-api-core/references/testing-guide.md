# Testing Guide

## 目录
- [测试 Middleware](#测试-middleware)
  - [基础模板](#基础模板)
  - [Mock Next Handler](#mock-next-handler)
  - [测试 Bag 交互](#测试-bag-交互)
  - [测试 Auth Middleware（完整示例）](#测试-auth-middleware完整示例)
- [测试 Plugin](#测试-plugin)
  - [基础 Plugin 测试](#基础-plugin-测试)
  - [测试 Transport 替换](#测试-transport-替换)
  - [测试事件监听](#测试事件监听)
  - [测试完整的 Plugin](#测试完整的-plugin)
- [测试 Request/Response 不可变性](#测试-requestresponse-不可变性)
- [测试错误场景](#测试错误场景)
  - [测试超时错误](#测试超时错误)
  - [测试中止错误](#测试中止错误)
- [集成测试](#集成测试)
  - [测试完整的请求链](#测试完整的请求链)
  - [测试 Plugin 和 Middleware 组合](#测试-plugin-和-middleware-组合)
- [测试最佳实践](#测试最佳实践)
  - [使用 Fake Timers](#使用-fake-timers)

## 测试 Middleware

### 基础模板

```ts
import { describe, expect, it } from "vitest";
import { Context, Request, Response, Middleware } from "@aptx/api-core";

describe("myMiddleware", () => {
  it("should add custom header", async () => {
    const mw: Middleware = {
      async handle(req, ctx, next) {
        const modifiedReq = req.with({
          headers: { "X-Custom": "value" }
        });
        return next(modifiedReq, ctx);
      }
    };

    const req = new Request({
      method: "GET",
      url: "https://example.com",
      headers: { "X-Base": "1" }
    });

    const ctx = new Context({
      id: "test-1",
      signal: new AbortController().signal
    });

    const next = async (r: Request, c: Context) => {
      return new Response({
        status: 200,
        headers: new Headers(),
        url: r.url,
        data: "ok",
        raw: {},
      });
    };

    const res = await mw.handle(req, ctx, next);

    // 验证请求被修改
    expect(req.headers.get("X-Base")).toBe("1");
    expect(req.headers.get("X-Custom")).toBe("value");

    // 验证响应正常
    expect(res.data).toBe("ok");
    expect(res.status).toBe(200);
  });
});
```

### Mock Next Handler

```ts
describe("retryMiddleware", () => {
  it("should retry on error", async () => {
    const mw = createRetryMiddleware({ retries: 2 });

    let attempt = 0;
    const next = async () => {
      attempt++;
      if (attempt === 1) {
        throw new NetworkError("Network error");
      }
      return new Response({
        status: 200,
        headers: new Headers(),
        url: "https://example.com",
        data: "ok",
        raw: {},
      });
    };

    const req = new Request({ method: "GET", url: "https://example.com" });
    const ctx = new Context({ id: "1", signal: new AbortController().signal });

    const res = await mw.handle(req, ctx, next);

    expect(attempt).toBe(2); // 初始请求 + 1 次重试
    expect(res.status).toBe(200);
  });

  it("should give up after max retries", async () => {
    const mw = createRetryMiddleware({ retries: 2 });

    let attempt = 0;
    const next = async () => {
      attempt++;
      throw new NetworkError("Network error");
    };

    const req = new Request({ method: "GET", url: "https://example.com" });
    const ctx = new Context({ id: "1", signal: new AbortController().signal });

    await expect(mw.handle(req, ctx, next)).rejects.toThrow(NetworkError);
    expect(attempt).toBe(3); // 初始请求 + 2 次重试
  });
});
```

### 测试 Bag 交互

```ts
import { createBagKey } from "@aptx/api-core";

describe("stateMiddleware", () => {
  it("should share state via bag", async () => {
    const STATE_KEY = createBagKey("state");

    const mw: Middleware = {
      async handle(req, ctx, next) {
        ctx.bag.set(STATE_KEY, "processing");

        try {
          const res = await next(req, ctx);
          ctx.bag.set(STATE_KEY, "completed");
          return res;
        } catch (err) {
          ctx.bag.set(STATE_KEY, "failed");
          throw err;
        }
      }
    };

    const req = new Request({ method: "GET", url: "https://example.com" });
    const ctx = new Context({ id: "1", signal: new AbortController().signal });

    const next = async (r: Request, c: Context) => {
      expect(c.bag.get(STATE_KEY)).toBe("processing");
      return new Response({
        status: 200,
        headers: new Headers(),
        url: r.url,
        data: "ok",
        raw: {},
      });
    };

    const res = await mw.handle(req, ctx, next);

    expect(ctx.bag.get(STATE_KEY)).toBe("completed");
  });

  it("should handle error case", async () => {
    const STATE_KEY = createBagKey("state");

    const mw: Middleware = {
      async handle(req, ctx, next) {
        ctx.bag.set(STATE_KEY, "processing");
        try {
          return await next(req, ctx);
        } catch (err) {
          ctx.bag.set(STATE_KEY, "failed");
          throw err;
        }
      }
    };

    const req = new Request({ method: "GET", url: "https://example.com" });
    const ctx = new Context({ id: "1", signal: new AbortController().signal });

    const next = async () => {
      throw new NetworkError("Network error");
    };

    await expect(mw.handle(req, ctx, next)).rejects.toThrow(NetworkError);
    expect(ctx.bag.get(STATE_KEY)).toBe("failed");
  });
});
```

### 测试 Auth Middleware（完整示例）

> 详见 [context-bag.md](context-bag.md) 了解 Auth Middleware 如何使用 Bag 防止无限重试，[middleware-patterns.md](middleware-patterns.md) 了解其他 Middleware 模式。

```ts
import { createAuthMiddleware } from "@aptx/api-plugin-auth";
import { HttpError } from "@aptx/api-core";

describe("authMiddleware", () => {
  it("should refresh on 401 and retry once", async () => {
    let token = "t0";
    const store: TokenStore = {
      getToken: () => token,
      setToken: (t) => { token = t; },
      clearToken: () => {},
    };

    const mw = createAuthMiddleware({
      store,
      refreshToken: async () => "t1",
      maxRetry: 1,
    });

    let call = 0;
    const next = async () => {
      call++;
      if (call === 1) {
        throw new HttpError("HTTP 401", 401, "https://example.com");
      }
      return new Response({
        status: 200,
        headers: new Headers(),
        url: "https://example.com",
        data: "ok",
        raw: {},
      });
    };

    const req = new Request({ method: "GET", url: "https://example.com" });
    const ctx = new Context({ id: "1", signal: new AbortController().signal });

    const res = await mw.handle(req, ctx, next);

    expect(res.data).toBe("ok");
    expect(token).toBe("t1");
    expect(call).toBe(2);
  });

  it("should call clearToken and onRefreshFailed when refresh fails", async () => {
    let cleared = false;
    let failed = false;
    const store: TokenStore = {
      getToken: () => "t0",
      setToken: () => {},
      clearToken: () => { cleared = true; },
    };

    const mw = createAuthMiddleware({
      store,
      refreshToken: async () => { throw new Error("Refresh failed"); },
      onRefreshFailed: (err) => { failed = true; },
    });

    const next = async () => {
      throw new HttpError("HTTP 401", 401, "https://example.com");
    };

    const req = new Request({ method: "GET", url: "https://example.com" });
    const ctx = new Context({ id: "1", signal: new AbortController().signal });

    await expect(mw.handle(req, ctx, next)).rejects.toThrow("Refresh failed");
    expect(cleared).toBe(true);
    expect(failed).toBe(true);
  });
});
```

---

## 测试 Plugin

### 基础 Plugin 测试

```ts
import { Plugin, Registry, Transport, TransportResult } from "@aptx/api-core";
import { RequestClient } from "@aptx/api-core";

describe("myPlugin", () => {
  it("should register middleware", async () => {
    let middlewareCalled = false;

    const testMiddleware: Middleware = {
      async handle(req, ctx, next) {
        middlewareCalled = true;
        return next(req, ctx);
      }
    };

    const plugin: Plugin = {
      setup(registry: Registry) {
        registry.use(testMiddleware);
      }
    };

    const client = new RequestClient();
    plugin.setup(client["registry"]());

    // 测试 middleware 是否被注册
    const pipeline = client["pipeline"];
    expect(pipeline).toBeDefined();
  });
});
```

### 测试 Transport 替换

```ts
describe("customTransportPlugin", () => {
  it("should replace transport", async () => {
    const mockTransport: Transport = {
      send: vi.fn().mockResolvedValue({
        status: 200,
        headers: new Headers(),
        url: "https://example.com",
        raw: {},
      })
    };

    const plugin: Plugin = {
      setup(registry: Registry) {
        registry.setTransport(mockTransport);
      }
    };

    const client = new RequestClient();
    plugin.setup(client["registry"]());

    const res = await client.fetch("https://example.com");

    expect(res.status).toBe(200);
    expect(mockTransport.send).toHaveBeenCalled();
  });
});
```

### 测试事件监听

```ts
describe("eventLoggingPlugin", () => {
  it("should listen to request events", async () => {
    const events: string[] = [];

    const plugin: Plugin = {
      setup(registry: Registry) {
        registry.events.on("request:start", () => {
          events.push("start");
        });

        registry.events.on("request:end", () => {
          events.push("end");
        });

        registry.events.on("request:error", () => {
          events.push("error");
        });
      }
    };

    const client = new RequestClient();
    plugin.setup(client["registry"]());

    const res = await client.fetch("https://example.com");

    expect(events).toEqual(["start", "end"]);
  });
});
```

### 测试完整的 Plugin

```ts
describe("myCompletePlugin", () => {
  it("should setup all components", async () => {
    const mockTransport: Transport = {
      send: vi.fn().mockResolvedValue({
        status: 200,
        headers: new Headers(),
        url: "https://example.com",
        raw: {},
      })
    };

    const plugin: Plugin = {
      setup(registry: Registry) {
        // 1. 替换 Transport
        registry.setTransport(mockTransport);

        // 2. 注册 middleware
        const testMiddleware: Middleware = {
          async handle(req, ctx, next) {
            return next(req.with({ url: "https://modified.com" }), ctx);
          }
        };
        registry.use(testMiddleware);

        // 3. 监听事件
        let eventFired = false;
        registry.events.on("request:end", () => {
          eventFired = true;
        });
      }
    };

    const client = new RequestClient();
    plugin.setup(client["registry"]());

    const res = await client.fetch("https://example.com");

    expect(res.status).toBe(200);
    expect(mockTransport.send).toHaveBeenCalledWith(
      expect.objectContaining({
        url: "https://modified.com" // middleware 修改了 URL
      }),
      expect.any(Context)
    );
  });
});
```

---

## 测试 Request/Response 不可变性

```ts
describe("Request/Response immutability", () => {
  it("does not allow external mutation of request headers", async () => {
    const req = new Request({
      method: "GET",
      url: "https://example.com",
      headers: { "X-Test": "1" },
    });

    const h1 = req.headers;
    h1.set("X-Test", "2");

    expect(req.headers.get("X-Test")).toBe("1"); // 原始请求未受影响
  });

  it("supports header removal in Request.with", async () => {
    const req = new Request({
      method: "GET",
      url: "https://example.com",
      headers: { "X-Test": "1", "X-Remove": "x" },
    });

    const next = async (r: Request) => {
      const modifiedReq = r.with({ headers: { "X-Remove": null } });
      return new Response({
        status: 200,
        headers: new Headers(),
        url: modifiedReq.url,
        data: "ok",
        raw: {},
      });
    };

    const res = await next(req);

    expect(req.headers.get("X-Test")).toBe("1");
    expect(req.headers.get("X-Remove")).toBe("x"); // 原始请求未受影响
  });

  it("does not allow external mutation of response headers", async () => {
    const res = new Response({
      status: 200,
      headers: new Headers({ "X-Resp": "1" }),
      url: "https://example.com",
      data: { ok: true },
      raw: {},
    });

    const h1 = res.headers;
    h1.set("X-Resp", "2");

    expect(res.headers.get("X-Resp")).toBe("1"); // 原始响应未受影响
  });
});
```

---

## 测试错误场景

### 测试超时错误

```ts
describe("timeout", () => {
  it("should timeout after specified duration", async () => {
    const slowTransport: Transport = {
      send: async () => {
        await new Promise(resolve => setTimeout(resolve, 5000));
        return {
          status: 200,
          headers: new Headers(),
          url: "https://example.com",
          raw: {},
        };
      }
    };

    const client = new RequestClient({
      transport: slowTransport,
      timeoutMs: 100,
    });

    await expect(client.fetch("https://example.com")).rejects.toThrow(TimeoutError);
  }, { timeout: 10000 });
});
```

### 测试中止错误

```ts
describe("abort", () => {
  it("should abort request when signal is aborted", async () => {
    const controller = new AbortController();
    const slowTransport: Transport = {
      send: async () => {
        await new Promise(resolve => setTimeout(resolve, 5000));
        return {
          status: 200,
          headers: new Headers(),
          url: "https://example.com",
          raw: {},
        };
      }
    };

    const client = new RequestClient({
      transport: slowTransport,
    });

    // 延迟中止
    setTimeout(() => controller.abort(), 100);

    await expect(
      client.fetch("https://example.com", { signal: controller.signal })
    ).rejects.toThrow(CanceledError);
  }, { timeout: 10000 });
});
```

---

## 集成测试

### 测试完整的请求链

```ts
describe("integration", () => {
  it("should work with multiple middlewares", async () => {
    const logs: string[] = [];

    const loggingMiddleware: Middleware = {
      async handle(req, ctx, next) {
        logs.push(`[1] ${req.method} ${req.url}`);
        const res = await next(req, ctx);
        logs.push(`[2] ${res.status}`);
        return res;
      }
    };

    const authMiddleware: Middleware = {
      async handle(req, ctx, next) {
        logs.push(`[3] adding auth header`);
        const authReq = req.with({
          headers: { "Authorization": "Bearer token" }
        });
        return next(authReq, ctx);
      }
    };

    const mockTransport: Transport = {
      send: async (req) => {
        expect(req.headers.get("Authorization")).toBe("Bearer token");
        return {
          status: 200,
          headers: new Headers(),
          url: req.url,
          raw: {},
        };
      }
    };

    const client = new RequestClient({ transport: mockTransport });
    client.use(loggingMiddleware);
    client.use(authMiddleware);

    const res = await client.fetch("https://example.com");

    expect(res.status).toBe(200);
    expect(logs).toEqual([
      "[1] GET https://example.com",
      "[3] adding auth header",
      "[2] 200"
    ]);
  });
});
```

### 测试 Plugin 和 Middleware 组合

```ts
describe("plugin + middleware integration", () => {
  it("should work with plugin and middleware together", async () => {
    const events: string[] = [];

    const eventLoggingPlugin: Plugin = {
      setup(registry) {
        registry.events.on("request:start", () => events.push("start"));
        registry.events.on("request:end", () => events.push("end"));
      }
    };

    const testMiddleware: Middleware = {
      async handle(req, ctx, next) {
        return next(req.with({ url: "https://modified.com" }), ctx);
      }
    };

    const mockTransport: Transport = {
      send: async (req) => {
        expect(req.url).toBe("https://modified.com");
        return {
          status: 200,
          headers: new Headers(),
          url: req.url,
          raw: {},
        };
      }
    };

    const client = new RequestClient({ transport: mockTransport });
    client.apply(eventLoggingPlugin);
    client.use(testMiddleware);

    const res = await client.fetch("https://example.com");

    expect(res.status).toBe(200);
    expect(events).toEqual(["start", "end"]);
  });
});
```

---

## 测试最佳实践

1. ✅ 使用 `Context` 的测试构造函数（不需要真实的 startTime）
2. ✅ Mock `next()` 函数控制测试场景
3. ✅ Mock `Transport` 避免真实网络请求
4. ✅ 验证 Request/Response 的不可变性
5. ✅ 测试错误场景（异常、超时、中止）
6. ✅ 测试 Bag 数据的传递和共享
7. ✅ 使用 `vi.fn()` 进行函数 mock（Vitest）
8. ❌ 不要依赖真实的网络请求（使用 Mock）
9. ❌ 不要在测试中使用真实的 `setTimeout`（使用 `vi.useFakeTimers()`）
10. ❌ 不要忽略错误场景的测试

### 使用 Fake Timers

```ts
describe("with fake timers", () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it("should handle timeout correctly", async () => {
    const slowTransport: Transport = {
      send: async () => {
        await new Promise(resolve => setTimeout(resolve, 5000));
        return {
          status: 200,
          headers: new Headers(),
          url: "https://example.com",
          raw: {},
        };
      }
    };

    const client = new RequestClient({
      transport: slowTransport,
      timeoutMs: 1000,
    });

    const promise = client.fetch("https://example.com");
    vi.advanceTimersByTime(1000);

    await expect(promise).rejects.toThrow(TimeoutError);
  });
});
```
