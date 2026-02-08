# TokenStore 实现示例

本文档提供 `TokenStore` 接口的各种实现示例。选择适合你场景的实现方式。

## 目录

- [最小实现（无过期控制）](#最小实现无过期控制)
- [完整实现（带过期管理）](#完整实现带过期管理)
- [Cookie 实现](#cookie-实现)
- [LocalStorage 实现](#localstorage-实现)
- [小程序实现](#小程序实现)
- [内存实现（仅用于测试）](#内存实现仅用于测试)

---

## 最小实现（无过期控制）

### 同步版本 - localStorage

```ts
const localStorageStore: TokenStore = {
  getToken: () => window.localStorage.getItem('token') || undefined,
  setToken: (token) => window.localStorage.setItem('token', token),
  clearToken: () => window.localStorage.removeItem('token'),
};
```

### 异步版本 - IndexedDB

```ts
const indexedDBStore: TokenStore = {
  async getToken() {
    const db = await openDB('aptx', 1);
    return await db.get('store', 'token');
  },
  async setToken(token) {
    const db = await openDB('aptx', 1);
    await db.put('store', { id: 'token', value: token });
  },
  async clearToken() {
    const db = await openDB('aptx', 1);
    await db.delete('store', 'token');
  },
};
```

---

## 完整实现（带过期管理）

```ts
const localStorageWithMetaStore: TokenStore = {
  getToken: () => window.localStorage.getItem('aptx_token') || undefined,

  setToken: (token, meta) => {
    window.localStorage.setItem('aptx_token', token);
    if (meta?.expiresAt) {
      window.localStorage.setItem('aptx_token_meta', JSON.stringify(meta));
    }
  },

  clearToken: () => {
    window.localStorage.removeItem('aptx_token');
    window.localStorage.removeItem('aptx_token_meta');
  },

  getMeta: () => {
    const raw = window.localStorage.getItem('aptx_token_meta');
    return raw ? JSON.parse(raw) : undefined;
  },

  setMeta: (meta) => {
    window.localStorage.setItem('aptx_token_meta', JSON.stringify(meta));
  },
};
```

---

## Cookie 实现

```ts
import Cookies from 'js-cookie';

class CookieTokenStore implements TokenStore {
  constructor(options: { tokenKey?: string; metaKey?: string; syncExpiryFromMeta?: boolean }) {
    this.tokenKey = options.tokenKey ?? 'aptx_token';
    this.metaKey = options.metaKey ?? 'aptx_token_meta';
    this.syncExpiryFromMeta = options.syncExpiryFromMeta ?? true;
  }

  private readonly tokenKey: string;
  private readonly metaKey: string;
  private readonly syncExpiryFromMeta: boolean;

  private cookieOptions(meta?: TokenMeta): Cookies.CookieAttributes {
    if (!this.syncExpiryFromMeta || !meta?.expiresAt) {
      return {};
    }
    return { expires: new Date(meta.expiresAt) };
  }

  getToken(): string | undefined {
    return Cookies.get(this.tokenKey);
  }

  setToken(token: string, meta?: TokenMeta): void {
    Cookies.set(this.tokenKey, token, this.cookieOptions(meta));
    if (meta) this.setMeta(meta);
  }

  clearToken(): void {
    Cookies.remove(this.tokenKey);
    Cookies.remove(this.metaKey);
  }

  getMeta(): TokenMeta | undefined {
    const raw = Cookies.get(this.metaKey);
    return raw ? JSON.parse(raw) : undefined;
  }

  setMeta(meta: TokenMeta): void {
    Cookies.set(this.metaKey, JSON.stringify(meta), this.cookieOptions(meta));
  }
}
```

---

## LocalStorage 实现

```ts
class LocalStorageTokenStore implements TokenStore {
  private readonly tokenKey = 'aptx_token';
  private readonly metaKey = 'aptx_token_meta';

  getToken(): string | undefined {
    return window.localStorage.getItem(this.tokenKey) || undefined;
  }

  setToken(token: string, meta?: TokenMeta): void {
    window.localStorage.setItem(this.tokenKey, token);
    if (meta) this.setMeta(meta);
  }

  clearToken(): void {
    window.localStorage.removeItem(this.tokenKey);
    window.localStorage.removeItem(this.metaKey);
  }

  getMeta(): TokenMeta | undefined {
    const raw = window.localStorage.getItem(this.metaKey);
    return raw ? JSON.parse(raw) : undefined;
  }

  setMeta(meta: TokenMeta): void {
    window.localStorage.setItem(this.metaKey, JSON.stringify(meta));
  }

  getRecord(): TokenRecord {
    return {
      token: this.getToken(),
      meta: this.getMeta(),
    };
  }

  setRecord(record: TokenRecord): void {
    if (record.token) {
      window.localStorage.setItem(this.tokenKey, record.token);
    } else {
      window.localStorage.removeItem(this.tokenKey);
    }

    if (record.meta) {
      window.localStorage.setItem(this.metaKey, JSON.stringify(record.meta));
    } else {
      window.localStorage.removeItem(this.metaKey);
    }
  }
}
```

---

## 小程序实现

```ts
class WeappTokenStore implements TokenStore {
  private readonly tokenKey = 'aptx_token';
  private readonly metaKey = 'aptx_token_meta';

  getToken(): string | undefined {
    return wx.getStorageSync(this.tokenKey) || undefined;
  }

  setToken(token: string, meta?: TokenMeta): void {
    wx.setStorageSync(this.tokenKey, token);
    if (meta) this.setMeta(meta);
  }

  clearToken(): void {
    wx.removeStorageSync(this.tokenKey);
    wx.removeStorageSync(this.metaKey);
  }

  getMeta(): TokenMeta | undefined {
    const raw = wx.getStorageSync(this.metaKey);
    return raw ? JSON.parse(raw) : undefined;
  }

  setMeta(meta: TokenMeta): void {
    wx.setStorageSync(this.metaKey, JSON.stringify(meta));
  }
}
```

---

## 内存实现（仅用于测试）

```ts
class MemoryTokenStore implements TokenStore {
  private token: string | undefined;
  private meta: TokenMeta | undefined;

  getToken(): string | undefined {
    return this.token;
  }

  setToken(token: string, meta?: TokenMeta): void {
    this.token = token;
    this.meta = meta;
  }

  clearToken(): void {
    this.token = undefined;
    this.meta = undefined;
  }

  getMeta(): TokenMeta | undefined {
    return this.meta;
  }

  setMeta(meta: TokenMeta): void {
    this.meta = meta;
  }

  getRecord(): TokenRecord {
    return { token: this.token, meta: this.meta };
  }

  setRecord(record: TokenRecord): void {
    this.token = record.token;
    this.meta = record.meta;
  }
}
```

---

## TokenStoreFactory 模式

当实现需要配置时，使用 Factory 模式：

```ts
import type { TokenStoreFactory } from '@aptx/token-store';

// 定义配置接口
export interface YourStoreOptions {
  tokenKey?: string;
  metaKey?: string;
  prefix?: string;
}

// 实现类
class YourTokenStore implements TokenStore {
  constructor(private options: YourStoreOptions) {
    this.tokenKey = options.tokenKey ?? 'aptx_token';
    this.metaKey = options.metaKey ?? 'aptx_token_meta';
    this.prefix = options.prefix ?? '';
  }

  private readonly tokenKey: string;
  private readonly metaKey: string;
  private readonly prefix: string;

  // ... 实现 TokenStore 方法
}

// Factory 函数
export const createYourTokenStore: TokenStoreFactory<YourStoreOptions>["create"] = (
  options: YourStoreOptions = {}
) => {
  return new YourTokenStore(options);
};

// 使用
const store = createYourTokenStore({
  tokenKey: 'my_token',
  metaKey: 'my_token_meta',
  prefix: 'myapp_',
});
```
