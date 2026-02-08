# 测试模式和实现注意事项

本文档提供 TokenStore 实现的测试模式和最佳实践。

## 目录

- [测试模式](#测试模式)
  - [Mock 存储](#mock-存储)
  - [使用内存存储进行测试](#使用内存存储进行测试)
  - [测试用例示例](#测试用例示例)
- [实现注意事项](#实现注意事项)
  - [错误处理](#错误处理)
  - [类型安全](#类型安全)
  - [性能考虑](#性能考虑)
  - [安全性](#安全性)

---

## 测试模式

### Mock 存储

```ts
const mockStore: TokenStore = {
  getToken: () => 'test-token',
  setToken: (token, meta) => {
    storedToken = token;
    storedMeta = meta;
  },
  clearToken: () => {
    storedToken = undefined;
    storedMeta = undefined;
  },
  getMeta: () => storedMeta,
};

// 验证集成
expect(storedToken).toBe('new-token');
expect(storedMeta?.expiresAt).toBeGreaterThan(Date.now());
```

### 使用内存存储进行测试

```ts
import { MemoryTokenStore } from './test-utils';

const store = new MemoryTokenStore();

client.use(createAuthMiddleware({
  store,
  refreshToken: async () => ({ token: 'test-token' }),
}));

// 测试后验证
expect(store.getToken()).toBe('test-token');
```

### 测试用例示例

#### 基本功能测试

```ts
describe('LocalStorageTokenStore', () => {
  let store: LocalStorageTokenStore;

  beforeEach(() => {
    localStorage.clear();
    store = new LocalStorageTokenStore();
  });

  describe('getToken', () => {
    it('should return undefined when no token is stored', () => {
      expect(store.getToken()).toBeUndefined();
    });

    it('should return the stored token', () => {
      store.setToken('test-token');
      expect(store.getToken()).toBe('test-token');
    });
  });

  describe('setToken', () => {
    it('should store the token', () => {
      store.setToken('test-token');
      expect(store.getToken()).toBe('test-token');
    });

    it('should store meta information', () => {
      const meta = { expiresAt: Date.now() + 3600 * 1000 };
      store.setToken('test-token', meta);
      expect(store.getMeta()).toEqual(meta);
    });
  });

  describe('clearToken', () => {
    it('should clear the stored token', () => {
      store.setToken('test-token');
      store.clearToken();
      expect(store.getToken()).toBeUndefined();
    });

    it('should also clear meta information', () => {
      const meta = { expiresAt: Date.now() + 3600 * 1000 };
      store.setToken('test-token', meta);
      store.clearToken();
      expect(store.getMeta()).toBeUndefined();
    });
  });
});
```

#### 过期控制测试

```ts
describe('Token expiration', () => {
  let store: LocalStorageTokenStore;

  beforeEach(() => {
    localStorage.clear();
    store = new LocalStorageTokenStore();
  });

  it('should store expiresAt in meta', () => {
    const expiresAt = Date.now() + 3600 * 1000;
    store.setToken('test-token', { expiresAt });
    expect(store.getMeta()?.expiresAt).toBe(expiresAt);
  });

  it('should handle past expiration', () => {
    const expiresAt = Date.now() - 3600 * 1000; // 已过期
    store.setToken('test-token', { expiresAt });
    expect(store.getMeta()?.expiresAt).toBeLessThan(Date.now());
  });
});
```

#### 记录方法测试

```ts
describe('TokenRecord', () => {
  let store: LocalStorageTokenStore;

  beforeEach(() => {
    localStorage.clear();
    store = new LocalStorageTokenStore();
  });

  it('getRecord should return both token and meta', () => {
    const token = 'test-token';
    const meta = { expiresAt: Date.now() + 3600 * 1000 };
    store.setToken(token, meta);

    const record = store.getRecord();
    expect(record.token).toBe(token);
    expect(record.meta).toEqual(meta);
  });

  it('setRecord should set both token and meta', () => {
    const record: TokenRecord = {
      token: 'test-token',
      meta: { expiresAt: Date.now() + 3600 * 1000 },
    };

    store.setRecord(record);
    expect(store.getToken()).toBe(record.token);
    expect(store.getMeta()).toEqual(record.meta);
  });

  it('setRecord should handle undefined token', () => {
    const record: TokenRecord = {
      token: undefined,
      meta: { expiresAt: Date.now() + 3600 * 1000 },
    };

    store.setToken('old-token');
    store.setRecord(record);
    expect(store.getToken()).toBeUndefined();
    expect(store.getMeta()).toEqual(record.meta);
  });
});
```

#### 同步/异步一致性测试

```ts
describe('Sync/Async consistency', () => {
  it('sync store should use sync methods', () => {
    const syncStore: TokenStore = {
      getToken: () => 'token',
      setToken: (t) => {},
      clearToken: () => {},
    };

    const result = syncStore.getToken();
    expect(typeof result).not.toBe('object'); // not a Promise
  });

  it('async store should use async methods', async () => {
    const asyncStore: TokenStore = {
      async getToken() {
        return 'token';
      },
      async setToken(t) {},
      async clearToken() {},
    };

    const result = await asyncStore.getToken();
    expect(result).toBe('token');
  });
});
```

---

## 实现注意事项

### 错误处理

#### LocalStorage 错误

```ts
class SafeLocalStorageTokenStore implements TokenStore {
  getToken(): string | undefined {
    try {
      return window.localStorage.getItem('aptx_token') || undefined;
    } catch (error) {
      console.error('Failed to read token from localStorage:', error);
      return undefined;
    }
  }

  setToken(token: string, meta?: TokenMeta): void {
    try {
      window.localStorage.setItem('aptx_token', token);
      if (meta) this.setMeta(meta);
    } catch (error) {
      console.error('Failed to write token to localStorage:', error);
      // 静默失败或抛出错误，取决于你的需求
    }
  }

  clearToken(): void {
    try {
      window.localStorage.removeItem('aptx_token');
      window.localStorage.removeItem('aptx_token_meta');
    } catch (error) {
      console.error('Failed to clear localStorage:', error);
    }
  }

  // ... 其他方法
}
```

#### 存储容量超限处理

```ts
// 检查 localStorage 可用空间
function checkLocalStorageSpace(): boolean {
  try {
    const testKey = '__localStorage_test__';
    localStorage.setItem(testKey, testKey);
    localStorage.removeItem(testKey);
    return true;
  } catch (error) {
    return false;
  }
}

// 在设置 token 前检查
setToken(token: string, meta?: TokenMeta): void {
  if (!checkLocalStorageSpace()) {
    console.warn('localStorage is not available');
    return;
  }
  // ... 继续设置
}
```

#### Cookie 禁用处理

```ts
class RobustCookieTokenStore implements TokenStore {
  private isCookieAvailable(): boolean {
    try {
      Cookies.set('__test__', 'test');
      Cookies.remove('__test__');
      return true;
    } catch {
      return false;
    }
  }

  getToken(): string | undefined {
    if (!this.isCookieAvailable()) {
      console.warn('Cookies are not available');
      return undefined;
    }
    return Cookies.get(this.tokenKey);
  }

  // ... 其他方法
}
```

### 类型安全

#### 使用完整类型定义

```ts
// 推荐做法
import type { TokenStore, TokenMeta, TokenRecord } from '@aptx/token-store';

class MyTokenStore implements TokenStore {
  getToken(): string | undefined {
    // ...
  }

  setToken(token: string, meta?: TokenMeta): void {
    // ...
  }

  // ...
}

// 不推荐
const badStore: any = {
  getToken: () => 'token',
  // 失去类型检查
};
```

#### 扩展类型（如需要）

```ts
// 继承基础类型
interface CustomTokenMeta extends TokenMeta {
  userId: string;
  roles: string[];
}

// 自定义 TokenRecord
interface CustomTokenRecord extends TokenRecord {
  meta?: CustomTokenMeta;
}

class CustomTokenStore implements TokenStore {
  private customMeta: CustomTokenMeta | undefined;

  setToken(token: string, meta?: CustomTokenMeta): void {
    this.customMeta = meta;
    // ...
  }
}
```

### 性能考虑

#### 避免阻塞主线程

```ts
// 同步实现应避免大量操作
class EfficientLocalStorageStore implements TokenStore {
  getToken(): string | undefined {
    // 直接读取，避免复杂计算
    return window.localStorage.getItem('aptx_token') || undefined;
  }

  setToken(token: string, meta?: TokenMeta): void {
    // 直接写入，避免大量数据序列化
    window.localStorage.setItem('aptx_token', token);
    if (meta?.expiresAt) {
      // 只存储必要的元数据
      window.localStorage.setItem('aptx_token_meta', JSON.stringify({
        expiresAt: meta.expiresAt,
      }));
    }
  }
}
```

#### 批量操作使用 getRecord/setRecord

```ts
// 低效：多次 I/O 操作
setToken(token, meta);
const newMeta = await store.getMeta();
newMeta.userId = '123';
await store.setMeta(newMeta);

// 高效：一次 I/O 操作
const record = store.getRecord();
record.meta = { ...record.meta, userId: '123' };
store.setRecord(record);
```

#### 并发控制（异步实现）

```ts
class AsyncTokenStore implements TokenStore {
  private operationQueue: Promise<void> = Promise.resolve();

  private async queueOperation<T>(operation: () => Promise<T>): Promise<T> {
    this.operationQueue = this.operationQueue.then(() => operation());
    return await this.operationQueue;
  }

  async getToken(): Promise<string | undefined> {
    return this.queueOperation(async () => {
      const db = await this.getDB();
      return await db.get('token');
    });
  }

  async setToken(token: string, meta?: TokenMeta): Promise<void> {
    return this.queueOperation(async () => {
      const db = await this.getDB();
      await db.put('token', token);
    });
  }
}
```

### 安全性

#### Cookie 安全配置

```ts
class SecureCookieTokenStore implements TokenStore {
  private cookieOptions(meta?: TokenMeta): Cookies.CookieAttributes {
    return {
      expires: meta?.expiresAt ? new Date(meta.expiresAt) : undefined,
      secure: true,           // 仅 HTTPS
      sameSite: 'lax',        // CSRF 防护
      httpOnly: false,        // 可通过 JS 访问（如果需要）
    };
  }

  setToken(token: string, meta?: TokenMeta): void {
    Cookies.set(this.tokenKey, token, this.cookieOptions(meta));
    // ...
  }
}
```

#### 敏感信息加密

```ts
import CryptoJS from 'crypto-js';

const SECRET_KEY = 'your-secret-key';

function encrypt(data: string): string {
  return CryptoJS.AES.encrypt(data, SECRET_KEY).toString();
}

function decrypt(data: string): string {
  return CryptoJS.AES.decrypt(data, SECRET_KEY).toString(CryptoJS.enc.Utf8);
}

class EncryptedLocalStorageStore implements TokenStore {
  getToken(): string | undefined {
    const encrypted = window.localStorage.getItem('aptx_token');
    if (!encrypted) return undefined;
    try {
      return decrypt(encrypted);
    } catch {
      return undefined;
    }
  }

  setToken(token: string, meta?: TokenMeta): void {
    window.localStorage.setItem('aptx_token', encrypt(token));
    // ...
  }
}
```

#### XSS 防护

```ts
// 验证 token 格式
function isValidToken(token: unknown): token is string {
  return typeof token === 'string' && token.length > 0;
}

class SecureLocalStorageStore implements TokenStore {
  getToken(): string | undefined {
    const token = window.localStorage.getItem('aptx_token');
    if (!token) return undefined;

    // 验证 token 格式
    if (!isValidToken(token)) {
      console.warn('Invalid token format, clearing...');
      this.clearToken();
      return undefined;
    }

    return token;
  }

  setToken(token: string, meta?: TokenMeta): void {
    if (!isValidToken(token)) {
      throw new Error('Invalid token format');
    }
    // ...
  }
}
```

#### 存储隔离

```ts
// 使用应用前缀避免冲突
class PrefixLocalStorageStore implements TokenStore {
  constructor(private prefix: string = 'aptx_') {
    if (!prefix.endsWith('_')) {
      this.prefix = prefix + '_';
    }
  }

  private tokenKey: string;
  private metaKey: string;

  constructor(prefix: string = 'myapp_') {
    this.tokenKey = `${prefix}token`;
    this.metaKey = `${prefix}token_meta`;
  }

  getToken(): string | undefined {
    return window.localStorage.getItem(this.tokenKey) || undefined;
  }
}
```
