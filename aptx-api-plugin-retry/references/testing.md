# 测试指南

## 基本测试用例

### 测试成功重试

```ts
import { describe, it, expect, vi } from 'vitest';
import { RequestClient } from '@aptx/api-core';
import { createRetryMiddleware } from '@aptx/api-plugin-retry';

describe('retry middleware', () => {
  it('should retry on network error and succeed', async () => {
    let attempts = 0;

    const mockTransport = {
      send: vi.fn(async () => {
        attempts++;
        if (attempts < 3) {
          const error = new Error('Network error');
          error.name = 'NetworkError';
          throw error;
        }
        return new Response({ status: 200, data: 'ok' });
      }),
    };

    const client = new RequestClient({
      transport: mockTransport,
    });

    client.use(createRetryMiddleware({
      retries: 3,
      delayMs: 0, // 测试时禁用延迟
    }));

    const res = await client.fetch('/test');
    expect(res.status).toBe(200);
    expect(attempts).toBe(3);
  });
});
```

### 测试放弃重试

```ts
it('should give up after max retries', async () => {
  let attempts = 0;

  const mockTransport = {
    send: vi.fn(async () => {
      attempts++;
      const error = new Error('Network error');
      error.name = 'NetworkError';
      throw error;
    }),
  };

  const client = new RequestClient({
    transport: mockTransport,
  });

  client.use(createRetryMiddleware({
    retries: 2,
    delayMs: 0,
  }));

  await expect(client.fetch('/test')).rejects.toThrow('Network error');
  expect(attempts).toBe(3); // 首次 + 2 次重试
});
```

### 测试 retryOn 条件

```ts
it('should only retry when retryOn returns true', async () => {
  let attempts = 0;

  const mockTransport = {
    send: vi.fn(async () => {
      attempts++;
      if (attempts === 1) {
        const error = new Error('Business error');
        error.status = 400;
        throw error;
      }
      return new Response({ status: 200, data: 'ok' });
    }),
  };

  const client = new RequestClient({
    transport: mockTransport,
  });

  client.use(createRetryMiddleware({
    retries: 3,
    delayMs: 0,
    retryOn: (error) => error.status >= 500, // 仅 5xx 重试
  }));

  await expect(client.fetch('/test')).rejects.toThrow('Business error');
  expect(attempts).toBe(1); // 不重试
});
```

## 延迟测试

### 测试延迟策略

```ts
it('should apply exponential backoff', async () => {
  vi.useFakeTimers();
  const delays: number[] = [];
  let attempts = 0;

  const mockTransport = {
    send: vi.fn(async () => {
      attempts++;
      if (attempts < 4) {
        throw Object.assign(new Error('Network error'), { name: 'NetworkError' });
      }
      return new Response({ status: 200 });
    }),
  };

  const client = new RequestClient({ transport: mockTransport });

  client.use(createRetryMiddleware({
    retries: 3,
    delayMs: (attempt) => {
      const delay = attempt * 1000;
      delays.push(delay);
      return delay;
    },
  }));

  const promise = client.fetch('/test');

  // 等待所有重试完成
  await vi.runAllTimersAsync();

  const res = await promise;
  expect(res.status).toBe(200);
  expect(delays).toEqual([1000, 2000, 3000]);

  vi.useRealTimers();
});
```

## ctx.attempt 测试

```ts
it('should track attempt count in ctx.attempt', async () => {
  const attempts: number[] = [];

  const client = new RequestClient();

  // 记录每次请求的 ctx.attempt
  client.use({
    handle: async (req, ctx, next) => {
      attempts.push(ctx.attempt);
      const result = await next(req, ctx);
      return result;
    },
  });

  client.use(createRetryMiddleware({
    retries: 2,
    delayMs: 0,
    retryOn: () => true,
  }));

  // 模拟失败后成功
  let callCount = 0;
  client.use({
    handle: async (req, ctx, next) => {
      callCount++;
      if (callCount < 3) {
        throw Object.assign(new Error('fail'), { name: 'NetworkError' });
      }
      return new Response({ status: 200 });
    },
  });

  await client.fetch('/test');
  expect(attempts).toEqual([0, 1, 2]); // 首次请求=0, 第1次重试=1, 第2次重试=2
});
```

## Per-call Override 测试

```ts
it('should respect per-call retry override', async () => {
  let attempts = 0;

  const mockTransport = {
    send: vi.fn(async () => {
      attempts++;
      throw Object.assign(new Error('Network error'), { name: 'NetworkError' });
    }),
  };

  const client = new RequestClient({ transport: mockTransport });

  client.use(createRetryMiddleware({
    retries: 3,
    delayMs: 0,
  }));

  // 禁用本次调用的重试
  await expect(
    client.fetch('/test', { meta: { __aptxRetry: { disable: true } } })
  ).rejects.toThrow();

  expect(attempts).toBe(1); // 仅首次请求，无重试
});
```

## Mock 工具函数

```ts
// 创建模拟网络错误的工具函数
export function createMockNetworkError(): Error {
  const error = new Error('Network error');
  error.name = 'NetworkError';
  return error;
}

// 创建模拟超时错误的工具函数
export function createMockTimeoutError(): Error {
  const error = new Error('Request timeout');
  error.name = 'TimeoutError';
  return error;
}

// 创建模拟 HTTP 错误的工具函数
export function createMockHttpError(status: number, message?: string): Error {
  const error = new Error(message || `HTTP ${status}`);
  error.name = 'HttpError';
  (error as any).status = status;
  return error;
}
```

## 测试覆盖清单

- [ ] 成功重试（网络错误后成功）
- [ ] 放弃重试（超过最大重试次数）
- [ ] retryOn 返回 false 时不重试
- [ ] 非幂等请求（POST/PUT）不重试
- [ ] 延迟策略正确执行
- [ ] ctx.attempt 正确递增
- [ ] Per-call override 生效
- [ ] 与其他 middleware 的顺序正确
- [ ] 错误正确传播（保留原始错误信息）
