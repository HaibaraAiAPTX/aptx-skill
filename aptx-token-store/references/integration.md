# 与 @aptx/api-plugin-auth 集成指南

本文档详细介绍如何将自定义 `TokenStore` 与 `@aptx/api-plugin-auth` 集成。

## 目录

- [基本集成模式](#基本集成模式)
- [api-plugin-auth 的关键调用](#api-plugin-auth-的关键调用)
- [集成要求](#集成要求)
- [常见集成场景](#常见集成场景)

---

## 基本集成模式

```ts
import { createAuthMiddleware } from '@aptx/api-plugin-auth';

client.use(createAuthMiddleware({
  store,                      // 你的 TokenStore 实现
  refreshLeewayMs: 60_000,   // 提前60秒刷新
  refreshToken: async () => {
    // 返回格式1：完整对象（推荐）
    return {
      token: 'new-token',
      expiresAt: Date.now() + 30 * 60 * 1000  // 30分钟后过期
    };

    // 或返回格式2：仅 token（不自动设置 expiresAt）
    // return 'new-token';
  },
}));
```

---

## api-plugin-auth 的关键调用

### 1. 主动刷新：在请求前检查过期

```ts
// api-plugin-auth 内部逻辑
const expiresAt = await store.getMeta()?.expiresAt;
if (expiresAt && Date.now() + refreshLeewayMs >= expiresAt) {
  // 调用 refreshToken 并设置新 token
  const result = await refreshToken();
  await store.setToken(result.token, { expiresAt: result.expiresAt });
}
```

### 2. 被动刷新：401 错误时刷新

```ts
// api-plugin-auth 内部逻辑
try {
  return await next();
} catch (error) {
  if (is401Error(error)) {
    const result = await refreshToken();
    await store.setToken(result.token, { expiresAt: result.expiresAt });
    return await next();  // 重试请求
  }
  throw error;
}
```

### 3. 失败处理：刷新失败时清除 token

```ts
// api-plugin-auth 内部逻辑
try {
  await refreshToken();
} catch (error) {
  await store.clearToken();
  if (onRefreshFailed) {
    onRefreshFailed(error);
  }
  throw error;
}
```

---

## 集成要求

### 必须实现的方法

- **`getToken()`** - 读取 token 用于请求
- **`setToken()`** - 保存刷新后的 token
- **`clearToken()`** - 刷新失败时清除无效 token

### 推荐实现的方法

- **`getMeta()`** - 支持 `expiresAt` 读取（主动刷新）
- **`setMeta()`** - 支持 `expiresAt` 写入

### 方法签名要求

```ts
interface TokenStore {
  // 核心方法（必须实现）
  getToken(): string | undefined | Promise<string | undefined>;
  setToken(token: string, meta?: TokenMeta): void | Promise<void>;
  clearToken(): void | Promise<void>;

  // 可选方法（支持过期控制）
  getMeta?(): TokenMeta | undefined | Promise<TokenMeta | undefined>;
  setMeta?(meta: TokenMeta): void | Promise<void>;
  getRecord?(): TokenRecord | undefined | Promise<TokenRecord | undefined>;
  setRecord?(record: TokenRecord): void | Promise<void>;
}
```

---

## 常见集成场景

### 场景 1：使用 localStorage 实现

```ts
import { createAuthMiddleware } from '@aptx/api-plugin-auth';
import { LocalStorageTokenStore } from './stores/localStorage';

const store = new LocalStorageTokenStore();

client.use(createAuthMiddleware({
  store,
  refreshLeewayMs: 60_000,
  refreshToken: async () => {
    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      credentials: 'include',
    });
    const data = await response.json();
    return {
      token: data.token,
      expiresAt: Date.now() + data.expiresIn * 1000,
    };
  },
}));
```

### 场景 2：使用 Cookie 实现（支持 SameSite）

```ts
import { createAuthMiddleware } from '@aptx/api-plugin-auth';
import { CookieTokenStore } from './stores/cookie';

const store = new CookieTokenStore({
  tokenKey: 'aptx_token',
  metaKey: 'aptx_token_meta',
  syncExpiryFromMeta: true,
});

client.use(createAuthMiddleware({
  store,
  refreshToken: async () => {
    // Token 刷新逻辑
    const { token, expiresIn } = await refreshAccessToken();
    return {
      token,
      expiresAt: Date.now() + expiresIn * 1000,
    };
  },
}));
```

### 场景 3：小程序实现

```ts
import { createAuthMiddleware } from '@aptx/api-plugin-auth';
import { WeappTokenStore } from './stores/weapp';

const store = new WeappTokenStore();

client.use(createAuthMiddleware({
  store,
  refreshLeewayMs: 30_000, // 提前30秒刷新
  refreshToken: async () => {
    const { code } = await wx.login();
    const res = await wx.request({
      url: 'https://api.example.com/auth/refresh',
      method: 'POST',
      data: { code },
    });
    return {
      token: res.data.token,
      expiresAt: Date.now() + res.data.expiresIn * 1000,
    };
  },
}));
```

### 场景 4：内存存储（仅用于测试）

```ts
import { createAuthMiddleware } from '@aptx/api-plugin-auth';
import { MemoryTokenStore } from './test-utils';

const store = new MemoryTokenStore();

client.use(createAuthMiddleware({
  store,
  refreshToken: async () => ({
    token: 'test-token',
    expiresAt: Date.now() + 3600 * 1000,
  }),
}));

// 测试后验证
expect(store.getToken()).toBe('test-token');
```

---

## 同步/异步集成注意事项

### 同步 Store 集成

```ts
// store 是同步的
const store: TokenStore = {
  getToken: () => localStorage.getItem('token'),
  // ...
};

// api-plugin-auth 会正确处理同步返回
client.use(createAuthMiddleware({
  store,
  refreshToken: async () => ({ token: '...' }),
}));
```

### 异步 Store 集成

```ts
// store 是异步的
const store: TokenStore = {
  async getToken() {
    const db = await openDB(...);
    return await db.get('token');
  },
  // ...
};

// api-plugin-auth 会正确处理 async/await
client.use(createAuthMiddleware({
  store,
  refreshToken: async () => ({ token: '...' }),
}));
```

### 重要原则

**所有方法必须保持同步/异步一致性，不可混用。**

```ts
// 错误示例：混用同步和异步
const badStore = {
  getToken: () => localStorage.getItem('token'),  // 同步
  async setToken(token: string) {  // 异步
    await db.put('token', token);
  },
};

// 正确示例：全部同步
const goodSyncStore = {
  getToken: () => localStorage.getItem('token'),
  setToken: (token: string) => localStorage.setItem('token', token),
};

// 正确示例：全部异步
const goodAsyncStore = {
  async getToken() {
    const db = await openDB(...);
    return await db.get('token');
  },
  async setToken(token: string) {
    const db = await openDB(...);
    await db.put('token', token);
  },
};
```

---

## 调试技巧

### 查看存储状态

```ts
// 在调试时检查 store 状态
console.log('Current token:', await store.getToken());
console.log('Token meta:', await store.getMeta());
console.log('Expires at:', (await store.getMeta())?.expiresAt);
console.log('Time until expiry:', (await store.getMeta())?.expiresAt - Date.now());
```

### 验证刷新逻辑

```ts
// 手动触发刷新测试
const middleware = createAuthMiddleware({
  store,
  refreshLeewayMs: 0, // 立即刷新
  refreshToken: async () => {
    console.log('Refresh triggered!');
    return { token: 'new-token', expiresAt: Date.now() + 3600 * 1000 };
  },
});

// 发送请求后观察控制台输出
client.use(middleware);
```

### 测试失败场景

```ts
// 测试刷新失败后的清除逻辑
const store = new MemoryTokenStore();
let refreshCount = 0;

const middleware = createAuthMiddleware({
  store,
  refreshToken: async () => {
    refreshCount++;
    if (refreshCount === 1) {
      throw new Error('Refresh failed');
    }
    return { token: 'new-token', expiresAt: Date.now() + 3600 * 1000 };
  },
});

// 第一次刷新失败，token 应该被清除
// expect(store.getToken()).toBe(undefined);
```
