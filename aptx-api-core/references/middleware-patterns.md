# Middleware Patterns

## 目录
- [核心结构](#核心结构)
- [常见模式](#常见模式)
  - [1. 日志 Middleware](#1-日志-middleware)
  - [2. 请求重写 Middleware](#2-请求重写-middleware)
  - [3. 请求头增强 Middleware](#3-请求头增强-middleware)
  - [4. 缓存 Middleware（简化版）](#4-缓存-middleware简化版)
  - [5. 请求签名 Middleware](#5-请求签名-middleware)
  - [6. 限流 Middleware](#6-限流-middleware)
  - [7. 请求去重 Middleware](#7-请求去重-middleware)
- [执行顺序](#执行顺序)
- [最佳实践](#最佳实践)

## 核心结构

```ts
import { Middleware, Request, Context, Response, Next } from "@aptx/api-core";

const myMiddleware: Middleware = {
  async handle(req: Request, ctx: Context, next: Next): Promise<Response> {
    // 1. 前置处理
    const modifiedReq = req.with({ /* ... */ });

    // 2. 调用下一个 handler
    const res = await next(modifiedReq, ctx);

    // 3. 后置处理
    return res;
  }
};
```

## 常见模式

### 1. 日志 Middleware

```ts
const loggingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const start = Date.now();
    console.log(`[${req.method}] ${req.url}`);

    try {
      const res = await next(req, ctx);
      const duration = Date.now() - start;
      console.log(`[${res.status}] ${req.url} (${duration}ms)`);
      return res;
    } catch (err) {
      const duration = Date.now() - start;
      console.log(`[ERROR] ${req.url} (${duration}ms)`, err);
      throw err;
    }
  }
};
```

### 2. 请求重写 Middleware

```ts
const apiVersionMiddleware: Middleware = {
  async handle(req, ctx, next) {
    if (req.url.startsWith("/api")) {
      const modifiedReq = req.with({
        url: `/v1${req.url}`
      });
      return next(modifiedReq, ctx);
    }
    return next(req, ctx);
  }
};
```

### 3. 请求头增强 Middleware

```ts
const userAgentMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const modifiedReq = req.with({
      headers: {
        "User-Agent": "MyApp/1.0",
      },
    });
    return next(modifiedReq, ctx);
  }
};
```

### 4. 缓存 Middleware（简化版）

> 详见 [context-bag.md](context-bag.md) 了解 Bag 的更多使用场景。

```ts
import { createBagKey } from "@aptx/api-core";

const CACHE_KEY = createBagKey("cache");

const cacheMiddleware: Middleware = {
  async handle(req, ctx, next) {
    if (req.method !== "GET") return next(req, ctx);

    const cache = ctx.bag.get(CACHE_KEY) as Map<string, unknown> || new Map();
    ctx.bag.set(CACHE_KEY, cache);

    const cached = cache.get(req.url);
    if (cached) return cached as Response;

    const res = await next(req, ctx);
    cache.set(req.url, res);
    return res;
  }
};
```

### 5. 请求签名 Middleware

```ts
const signatureMiddleware: Middleware = {
  async handle(req, ctx, next) {
    // 生成签名（示例）
    const signature = btoa(`${req.method}${req.url}${req.body || ''}`);

    const modifiedReq = req.with({
      headers: {
        "X-Signature": signature,
      },
    });
    return next(modifiedReq, ctx);
  }
};
```

### 6. 限流 Middleware

```ts
import { createBagKey } from "@aptx/api-core";

const RATE_LIMIT_KEY = createBagKey("rate_limit");

const rateLimitMiddleware = (maxRequests: number, windowMs: number): Middleware => {
  return {
    async handle(req, ctx, next) {
      let requests = (ctx.bag.get(RATE_LIMIT_KEY) as number) || 0;

      if (requests >= maxRequests) {
        throw new Error("Rate limit exceeded");
      }

      ctx.bag.set(RATE_LIMIT_KEY, requests + 1);
      return next(req, ctx);
    }
  };
};
```

### 7. 请求去重 Middleware

```ts
const pendingRequests = new Map<string, Promise<Response>>();

const dedupeMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const key = `${req.method}:${req.url}`;

    // 如果有相同请求正在进行，返回相同的 Promise
    if (pendingRequests.has(key)) {
      return pendingRequests.get(key)!;
    }

    // 创建新的请求 Promise
    const promise = next(req, ctx);
    pendingRequests.set(key, promise);

    // 请求完成后清除
    return promise.finally(() => {
      pendingRequests.delete(key);
    });
  }
};
```

## 执行顺序

Middlewares 按照 `use()` 的顺序执行：

```ts
client.use(loggingMiddleware);      // 第 1 个执行
client.use(authenticationMiddleware);  // 第 2 个执行
client.use(retryMiddleware);        // 第 3 个执行

// 执行流程：
// 1. loggingMiddleware 前置
// 2. authenticationMiddleware 前置
// 3. retryMiddleware 前置
// 4. actual request
// 5. retryMiddleware 后置
// 6. authenticationMiddleware 后置
// 7. loggingMiddleware 后置
```

## 最佳实践

1. ✅ 修改 Request 时使用 `req.with()` 而非直接修改
2. ✅ 错误应该抛出，不应在 middleware 中吞没
3. ✅ 前置处理在 `next()` 前，后置处理在 `next()` 后
4. ✅ 后置处理用于日志、缓存、监控等场景
5. ✅ 使用 `try/catch` 捕获 `next()` 的错误
6. ❌ 不要忘记调用 `next()` 或 `return next(...)`
7. ❌ 不要阻塞 pipeline（长时间同步操作）
8. ❌ 不要修改 Response 对象（Response 是不可变的）
