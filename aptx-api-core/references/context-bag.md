# Context and Bag

## 目录
- [Context 结构](#context-结构)
- [Bag 的用途](#bag-的用途)
- [创建 Bag Key](#创建-bag-key)
- [TIMEOUT_BAG_KEY - 超时检测](#timeout_bag_key---超时检测)
- [类型安全检查](#类型安全检查)
- [使用示例](#使用示例)
  - [1. 跟踪请求状态](#1-跟踪请求状态)
  - [2. 防止无限循环（参考 auth middleware）](#2-防止无限循环参考-auth-middleware)
  - [3. 共享缓存实例](#3-共享缓存实例)
  - [4. 请求计器](#4-请求计器)
  - [5. 性能追踪](#5-性能追踪)
  - [6. 跨 Middleware 数据传递](#6-跨-middleware-数据传递)
  - [7. 请求 ID 追踪](#7-请求-id-追踪)
- [Context 内置字段](#context-内置字段)
  - [ctx.id - 请求唯一标识](#ctxid---请求唯一标识)
  - [ctx.attempt - 重试次数](#ctxattempt---重试次数)
  - [ctx.startTime - 请求开始时间](#ctxstarttime---请求开始时间)
  - [ctx.signal - 中止信号](#ctxsignal---中止信号)
- [最佳实践](#最佳实践-5)
- [实际案例：Auth Middleware 中的 Bag 使用](#实际案例auth-middleware-中的-bag-使用)

## Context 结构

> 详见 [plugin-patterns.md](plugin-patterns.md) 了解 Plugin 如何使用 Context 和 Bag，以及 [middleware-patterns.md](middleware-patterns.md) 了解 Middleware 如何在 Bag 中共享状态。

```typescript
interface Context {
  id: string;              // 请求唯一 ID
  signal: AbortSignal;      // 中止信号
  bag: Map<symbol, unknown>; // 中间件间共享状态
  attempt: number;          // 重试次数（由 retry middleware 更新）
  startTime: number;        // 请求开始时间
}
```

## Bag 的用途

`ctx.bag` 是 middleware/plugin 之间传递数据的**安全机制**：

- 使用 `symbol` 作为 key，避免命名冲突
- 避免污染 Request/Response 对象
- 支持跨 middleware 共享状态
- 请求作用域的数据存储（每个请求独立的 bag）

## 创建 Bag Key

```ts
import { createBagKey } from "@aptx/api-core";

// 推荐：使用 createBagKey 创建唯一 symbol
const RETRY_COUNT_KEY = createBagKey("retry_count");
const CACHE_KEY = createBagKey("cache");
const AUTH_STATE_KEY = createBagKey("auth_state");
const LOGGING_KEY = createBagKey("logging_data");
```

## TIMEOUT_BAG_KEY - 超时检测

默认 bag key，用于区分超时和用户取消：

```ts
import { TIMEOUT_BAG_KEY } from "@aptx/api-core";

const timeoutCheckingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    try {
      return await next(req, ctx);
    } catch (err) {
      if (ctx.bag.get(TIMEOUT_BAG_KEY) === true) {
        console.log("Request timed out");
      } else {
        console.log("Request was cancelled by user");
      }
      throw err;
    }
  }
};
```

## 类型安全检查

使用 `assertBagKey` 确保 bag key 是 symbol 类型：

```ts
import { assertBagKey } from "@aptx/api-core";

function getFromBag<T>(ctx: Context, key: unknown): T | undefined {
  assertBagKey(key); // 如果不是 symbol 会抛出 ConfigError
  return ctx.bag.get(key) as T;
}
```

## 使用示例

### 1. 跟踪请求状态

```ts
import { Middleware, createBagKey } from "@aptx/api-core";

const STATE_KEY = createBagKey("request_state");

const stateTrackingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    ctx.bag.set(STATE_KEY, { phase: "processing" });

    try {
      const res = await next(req, ctx);
      ctx.bag.set(STATE_KEY, { phase: "completed" });
      return res;
    } catch (err) {
      ctx.bag.set(STATE_KEY, { phase: "failed", error: err });
      throw err;
    }
  }
};

// 另一个 middleware 读取状态
const loggingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const state = ctx.bag.get(STATE_KEY) as { phase: string } | undefined;
    console.log(`Current phase: ${state?.phase || "unknown"}`);
    return next(req, ctx);
  }
};
```

### 2. 防止无限循环（参考 auth middleware）

```ts
import { Middleware, createBagKey, HttpError } from "@aptx/api-core";

const AUTH_RETRY_KEY = createBagKey("auth_retry");

const authMiddleware: Middleware = {
  async handle(req, ctx, next) {
    try {
      return await next(req, ctx);
    } catch (err) {
      const error = err as Error;
      const shouldRefresh = error instanceof HttpError && error.status === 401;

      if (!shouldRefresh) throw error;

      const currentRetry = (ctx.bag.get(AUTH_RETRY_KEY) as number) ?? 0;
      if (currentRetry >= 1) throw error; // 防止无限重试

      ctx.bag.set(AUTH_RETRY_KEY, currentRetry + 1);

      // 刷新 token 并重试
      const newToken = await refreshToken();
      const retryReq = req.with({
        headers: { "Authorization": `Bearer ${newToken}` }
      });
      return await next(retryReq, ctx);
    }
  }
};
```

### 3. 共享缓存实例

```ts
import { Middleware, createBagKey } from "@aptx/api-core";

const CACHE_KEY = createBagKey("request_cache");

const cacheMiddleware: Middleware = {
  async handle(req, ctx, next) {
    if (req.method !== "GET") return next(req, ctx);

    let cache = ctx.bag.get(CACHE_KEY) as Map<string, unknown>;
    if (!cache) {
      cache = new Map();
      ctx.bag.set(CACHE_KEY, cache);
    }

    const cached = cache.get(req.url);
    if (cached) return cached as Response;

    const res = await next(req, ctx);
    cache.set(req.url, res);
    return res;
  }
};
```

### 4. 请求计器

```ts
import { Middleware, createBagKey } from "@aptx/api-core";

const COUNTER_KEY = createBagKey("request_counter");

const counterMiddleware: Middleware = {
  async handle(req, ctx, next) {
    let count = (ctx.bag.get(COUNTER_KEY) as number) ?? 0;
    ctx.bag.set(COUNTER_KEY, count + 1);

    console.log(`Request #${count + 1} for ${ctx.id}`);

    return next(req, ctx);
  }
};

// 在最后读取总次数
const finalLoggingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const res = await next(req, ctx);
    const count = ctx.bag.get(COUNTER_KEY) as number;
    console.log(`Total attempts: ${count}`);
    return res;
  }
};
```

### 5. 性能追踪

```ts
import { Middleware, createBagKey } from "@aptx/api-core";

const TIMING_KEY = createBagKey("timing");

const timingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const timings: { [key: string]: number } = {};

    const start = Date.now();

    // 中间件 1 开始
    timings.middleware1_start = Date.now();
    timings.middleware1_end = Date.now();

    // 调用下一个
    const res = await next(req, ctx);

    const end = Date.now();
    timings.total = end - start;

    ctx.bag.set(TIMING_KEY, timings);

    return res;
  }
};

// Plugin 监听事件并读取 timings
const timingPlugin: Plugin = {
  setup(registry: Registry) {
    registry.events.on("request:end", ({ req, res, ctx }) => {
      const timings = ctx.bag.get(TIMING_KEY) as { [key: string]: number } | undefined;
      if (timings) {
        console.log(`Request timings:`, timings);
      }
    });
  }
};
```

### 6. 跨 Middleware 数据传递

```ts
import { Middleware, createBagKey } from "@aptx/api-core";

const USER_KEY = createBagKey("current_user");

const userFetchingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    // 从请求头获取用户 ID
    const userId = req.headers.get("X-User-ID");

    if (userId) {
      // 假设这里有一个获取用户的方法
      const user = await fetchUser(userId);
      ctx.bag.set(USER_KEY, user);
    }

    return next(req, ctx);
  }
};

const userValidationMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const user = ctx.bag.get(USER_KEY);

    if (!user) {
      throw new Error("User not authenticated");
    }

    // 验证用户权限
    if (!user.isAdmin && req.url.startsWith("/admin")) {
      throw new Error("Permission denied");
    }

    return next(req, ctx);
  }
};
```

### 7. 请求 ID 追踪

```ts
import { Middleware, createBagKey } from "@aptx/api-core";

const TRACE_ID_KEY = createBagKey("trace_id");

const traceIdMiddleware: Middleware = {
  async handle(req, ctx, next) {
    // 从请求头获取 trace ID
    const traceId = req.headers.get("X-Trace-ID") || ctx.id;

    ctx.bag.set(TRACE_ID_KEY, traceId);

    // 所有日志都带上 trace ID
    console.log(`[${traceId}] ${req.method} ${req.url}`);

    return next(req, ctx);
  }
};

const errorLoggingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    try {
      return await next(req, ctx);
    } catch (err) {
      const traceId = ctx.bag.get(TRACE_ID_KEY) as string;
      console.error(`[${traceId}] Error:`, err);
      throw err;
    }
  }
};
```

## Context 内置字段

### ctx.id - 请求唯一标识

```ts
const loggingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    console.log(`[${ctx.id}] Starting request`);
    const res = await next(req, ctx);
    console.log(`[${ctx.id}] Completed`);
    return res;
  }
};
```

### ctx.attempt - 重试次数

由 retry middleware 自动更新：

```ts
const attemptLoggingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    console.log(`Attempt: ${ctx.attempt}`);
    return next(req, ctx);
  }
};
```

### ctx.startTime - 请求开始时间

```ts
const durationLoggingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const res = await next(req, ctx);
    const duration = Date.now() - ctx.startTime;
    console.log(`Duration: ${duration}ms`);
    return res;
  }
};
```

### ctx.signal - 中止信号

```ts
const cancellationMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const onAbort = () => {
      console.log(`Request ${ctx.id} was aborted`);
    };

    ctx.signal.addEventListener("abort", onAbort);

    try {
      return await next(req, ctx);
    } finally {
      ctx.signal.removeEventListener("abort", onAbort);
    }
  }
};
```

## 最佳实践

1. ✅ 始终使用 `createBagKey()` 创建 key
2. ✅ 使用描述性的 key 名称（如 `"auth_retry"`、`"user_data"`）
3. ✅ Bag 数据应该是请求作用域的（不要跨请求共享）
4. ✅ 在 bag 中存储轻量级数据（避免性能问题）
5. ✅ 使用 TypeScript 类型断言：`ctx.bag.get(KEY) as Type`
6. ❌ 不要使用字符串作为 bag key（容易冲突）
7. ❌ 不要在 bag 中存储大型对象（影响性能）
8. ❌ 不要假设 bag 中一定有某个 key（使用 `??` 提供默认值）
9. ❌ 不要在 bag 中存储跨请求的数据（使用外部存储）

## 实际案例：Auth Middleware 中的 Bag 使用

参考 `@aptx/api-plugin-auth` 的实现：

```ts
const AUTH_RETRY_KEY = createBagKey("auth_retry");

export function createAuthMiddleware(options: AuthPluginOptions): Middleware {
  return {
    async handle(req, ctx, next) {
      // ... 认证逻辑

      try {
        return await next(authedReq, ctx);
      } catch (err) {
        const shouldRefresh = options.shouldRefresh
          ? options.shouldRefresh(error, authedReq, ctx)
          : defaultShouldRefresh(error);

        if (!shouldRefresh) throw error;

        // 使用 bag 防止无限重试
        const currentRetry = (ctx.bag.get(AUTH_RETRY_KEY) as number | undefined) ?? 0;
        if (currentRetry >= maxRetry) throw error;

        ctx.bag.set(AUTH_RETRY_KEY, currentRetry + 1);

        // 刷新 token 并重试
        const newToken = await controller.refresh();
        const retryReq = authedReq.with({
          headers: { [headerName]: `${tokenPrefix}${newToken}` }
        });
        return await next(retryReq, ctx);
      }
    },
  };
};
```

这个例子展示了 bag 的核心用途：**在多次调用之间传递状态，防止无限循环**。
