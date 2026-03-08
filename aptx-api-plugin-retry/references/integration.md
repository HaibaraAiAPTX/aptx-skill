# 插件集成

## Middleware 顺序

重试中间件的位置非常重要。推荐顺序：

```ts
const client = new RequestClient();

// 1. 日志/监控（最外层，能观测所有行为）
client.use(loggingMiddleware);

// 2. 重试（在认证之前，避免认证失败时重试）
client.use(retryMiddleware);

// 3. 认证（在重试之后，确保每次重试都带正确的 token）
client.use(authMiddleware);

// 4. CSRF（在认证之后）
client.use(csrfMiddleware);

// 5. 缓存（最内层，避免缓存层重试）
client.use(cacheMiddleware);
```

## 与 @aptx/api-plugin-auth 集成

### 问题：认证失败时的重试

如果 auth middleware 返回 401，retry middleware 可能会不断重试。解决方案：

```ts
import { createRetryMiddleware } from '@aptx/api-plugin-retry';
import { createAuthMiddleware } from '@aptx/api-plugin-auth';

const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => attempt * 1000,
  retryOn: (error, req, ctx) => {
    // 401 不重试，由 auth middleware 处理
    if (error.status === 401) {
      return false;
    }
    return error.name === 'NetworkError' || error.name === 'TimeoutError';
  },
});

const auth = createAuthMiddleware({
  store,
  refreshToken: async () => { /* ... */ },
});

// 顺序很重要：retry 在 auth 之前
client.use(retry);
client.use(auth);
```

### Auth 刷新请求不应触发重试

当 auth middleware 刷新 token 时，刷新请求本身不应被重试：

```ts
import { SKIP_AUTH_REFRESH_META_KEY } from '@aptx/api-plugin-auth';

const refreshToken = async () => {
  // 刷新请求跳过重试
  const res = await client.fetch('/api/refresh', {
    method: 'POST',
    meta: {
      [SKIP_AUTH_REFRESH_META_KEY]: true,
      __aptxRetry: { disable: true }, // 同时禁用重试
    },
  });
  return res.data;
};
```

## 与 @aptx/api-plugin-csrf 集成

CSRF token 应该在每次请求时读取，不受重试影响：

```ts
import { createCsrfMiddleware } from '@aptx/api-plugin-csrf';

const csrf = createCsrfMiddleware({
  cookieName: 'XSRF-TOKEN',
  headerName: 'X-XSRF-TOKEN',
});

// CSRF 在 auth 之后，确保每次重试都有正确的 CSRF token
client.use(retry);
client.use(auth);
client.use(csrf);
```

## 与缓存 Middleware 集成

缓存应该在重试之前，避免缓存穿透：

```ts
const cacheMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const cacheKey = `${req.method}:${req.url}`;
    const cached = cache.get(cacheKey);

    if (cached) {
      return cached;
    }

    const res = await next(req, ctx);

    // 仅缓存成功响应
    if (res.status === 200) {
      cache.set(cacheKey, res, { ttl: 60000 });
    }

    return res;
  },
};

// 缓存在重试之后，确保重试成功后也能缓存
client.use(cacheMiddleware);
client.use(retry);
```

## 幂等性控制

### 自动检测非幂等请求

```ts
const IDEMPOTENT_METHODS = ['GET', 'HEAD', 'OPTIONS', 'TRACE'];

const retry = createRetryMiddleware({
  retries: 3,
  retryOn: (error, req, ctx) => {
    // 非幂等方法不重试
    if (!IDEMPOTENT_METHODS.includes(req.method)) {
      return false;
    }
    return error.name === 'NetworkError' || error.name === 'TimeoutError';
  },
});
```

### 允许特定 POST 重试

某些 POST 请求是幂等的（如幂等键模式）：

```ts
const retry = createRetryMiddleware({
  retries: 3,
  retryOn: (error, req, ctx) => {
    // 带有幂等键的请求可以重试
    const hasIdempotencyKey = req.headers['Idempotency-Key'];
    if (req.method === 'POST' && hasIdempotencyKey) {
      return true;
    }
    // 其他 POST 不重试
    if (req.method === 'POST') {
      return false;
    }
    return error.name === 'NetworkError';
  },
});
```

## 多实例场景

### 不同 API 使用不同重试策略

```ts
// 核心 API - 激进重试
const coreClient = new RequestClient({ baseURL: '/api/core' });
coreClient.use(createRetryMiddleware({
  retries: 5,
  delayMs: (attempt) => Math.pow(2, attempt) * 1000,
  retryOn: (error) => error.name === 'NetworkError',
}));

// 支付 API - 不重试
const paymentClient = new RequestClient({ baseURL: '/api/payment' });
// 不添加重试中间件

// 日志 API - 轻量重试
const logClient = new RequestClient({ baseURL: '/api/logs' });
logClient.use(createRetryMiddleware({
  retries: 1,
  delayMs: 100,
  retryOn: (error) => error.name === 'NetworkError',
}));
```

## 错误追踪

### 结合事件系统追踪重试

```ts
const client = new RequestClient();

client.events.on('request:error', ({ req, error, ctx, attempt }) => {
  if (attempt > 0) {
    analytics.track('api_retry', {
      url: req.url,
      method: req.method,
      attempt,
      error: error.message,
    });
  }
});

client.use(createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => attempt * 1000,
}));
```

## SSR 环境注意事项

SSR 环境下重试需要特别处理：

```ts
const isServer = typeof window === 'undefined';

const retry = createRetryMiddleware({
  // SSR 环境减少重试次数，避免阻塞渲染
  retries: isServer ? 1 : 3,
  delayMs: isServer ? 100 : (attempt) => attempt * 1000,
  retryOn: (error, req, ctx) => {
    // SSR 环境仅对特定错误重试
    if (isServer) {
      return error.name === 'NetworkError';
    }
    return error.name === 'NetworkError' || error.name === 'TimeoutError';
  },
});
```
