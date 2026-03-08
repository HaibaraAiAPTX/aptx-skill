# 延迟策略

## 固定延迟

最简单的延迟策略，每次重试使用相同的延迟时间：

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: 1000, // 每次重试前等待 1 秒
});
```

## 线性递增延迟

每次重试延迟时间线性增加：

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => attempt * 500, // 第1次: 500ms, 第2次: 1000ms, 第3次: 1500ms
});
```

## 指数退避

延迟时间按指数增长，是生产环境推荐的策略：

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => Math.min(1000 * Math.pow(2, attempt), 30000), // 1s, 2s, 4s, 最大 30s
});
```

## 指数退避 + 抖动（推荐）

添加随机抖动避免多个客户端同时重试（惊群效应）：

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => {
    const baseDelay = Math.min(1000 * Math.pow(2, attempt), 30000);
    const jitter = Math.random() * 1000; // 0-1000ms 随机抖动
    return baseDelay + jitter;
  },
});
```

### 全抖动（Full Jitter）

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => {
    const baseDelay = Math.min(1000 * Math.pow(2, attempt), 30000);
    return Math.random() * baseDelay; // 0 到 baseDelay 之间的随机值
  },
});
```

### 去相关抖动（Decorrelated Jitter）

```ts
let lastDelay = 0;

const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => {
    const base = 500;
    const cap = 30000;
    const sleep = Math.min(cap, Math.random() * (lastDelay * 3) + base);
    lastDelay = sleep;
    return sleep;
  },
});
```

## 基于错误类型的延迟

根据错误类型决定延迟时间：

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt, error, req, ctx) => {
    // 超时错误使用更长延迟
    if (error.name === 'TimeoutError') {
      return attempt * 2000;
    }
    // 网络错误使用较短延迟
    if (error.name === 'NetworkError') {
      return attempt * 500;
    }
    // 默认延迟
    return attempt * 1000;
  },
});
```

## 基于请求上下文的延迟

根据请求信息决定延迟：

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt, error, req, ctx) => {
    // 重要请求（带 tag）使用更长延迟
    if (req.meta?.tags?.includes('critical')) {
      return attempt * 3000;
    }
    // 普通请求
    return attempt * 1000;
  },
});
```

## 生产环境推荐配置

```ts
const retry = createRetryMiddleware({
  retries: 3,
  delayMs: (attempt) => {
    const baseDelay = 1000 * Math.pow(2, attempt);
    const cap = 30000;
    const jitter = Math.random() * 1000;
    return Math.min(baseDelay, cap) + jitter;
  },
  retryOn: (error, req, ctx) => {
    // 仅对网络错误和 5xx 重试
    return (
      error.name === 'NetworkError' ||
      error.name === 'TimeoutError' ||
      (error.status >= 500 && error.status < 600)
    );
  },
});
```
