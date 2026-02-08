# API Details

## CsrfOptions

```typescript
interface CsrfOptions {
  /**
   * CSRF cookie 名称
   * @default "XSRF-TOKEN"
   */
  cookieName?: string;

  /**
   * 写入 CSRF token 的 header 名称
   * @default "X-XSRF-TOKEN"
   */
  headerName?: string;

  /**
   * 仅在同源请求中附加 token
   * - true: 仅当请求 URL 匹配 window.location.origin 时才添加 token
   * - false: 无论 origin 如何都添加 token
   * @default true
   */
  sameOriginOnly?: boolean;

  /**
   * 自定义 cookie 读取函数
   * - 浏览器: 默认使用 document.cookie
   * - SSR/Node: 必须提供自定义实现
   *
   * @param name - 要读取的 cookie 名称
   * @returns cookie 值（未找到返回 undefined）
   */
  getCookie?: (name: string) => string | undefined;
}
```

## createCsrfMiddleware

```typescript
function createCsrfMiddleware(options?: CsrfOptions): Middleware
```

## 配置选项详解

### cookieName

CSRF token 在 cookie 中的名称。必须与后端设置的 cookie 名称匹配。

**默认值:** `"XSRF-TOKEN"`

```ts
createCsrfMiddleware({
  cookieName: "XSRF-TOKEN",  // 与后端约定一致
});
```

### headerName

将 CSRF token 写入到请求头的 header 名称。后端会从该 header 读取 token。

**默认值:** `"X-XSRF-TOKEN"`

```ts
createCsrfMiddleware({
  headerName: "X-XSRF-TOKEN",  // 后端从该 header 读取
});
```

### sameOriginOnly

控制是否只在同源请求中添加 CSRF token。

**默认值:** `true`

**行为差异:**

| 值 | 同源请求 | 跨域请求 |
|---|---------|---------|
| `true` | ✅ 添加 token | ❌ 不添加 |
| `false` | ✅ 添加 token | ✅ 添加 token |

**安全考虑:**

- ✅ 默认 `true` 对大多数用例是安全的
- ⚠️ 仅在明确需要跨域 CSRF token 时设置 `false`
- ⚠️ 跨域场景可能导致 token 泄露

```ts
// 仅同源请求添加 token（推荐）
createCsrfMiddleware({
  sameOriginOnly: true,
});

// 所有请求都添加 token（谨慎使用）
createCsrfMiddleware({
  sameOriginOnly: false,
});
```

### getCookie

自定义 cookie 读取函数。

**默认行为:**
- 浏览器环境: 自动使用 `document.cookie`
- SSR/Node 环境: 必须提供自定义实现，否则会报错

**函数签名:**
```ts
getCookie: (name: string) => string | undefined
```

**返回值:**
- `string`: 找到 cookie，返回其值
- `undefined`: 未找到 cookie

**示例:**

```ts
// 自定义 cookie 读取逻辑
createCsrfMiddleware({
  getCookie: (name) => {
    // 你的自定义实现
    return customCookieStore.get(name);
  }
});
```

## 默认实现细节

### 默认 getCookie（浏览器）

```ts
function defaultGetCookie(name: string): string | undefined {
  const cookies = document.cookie;
  const pattern = new RegExp(`(?:^|; )${name.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")}=([^;]*)`);
  const match = cookies.match(pattern);
  const value = match?.[1];
  return value ? decodeURIComponent(value) : undefined;
}
```

### 默认 isSameOrigin

```ts
function defaultIsSameOrigin(requestUrl: string): boolean {
  try {
    // SSR/Node 环境始终返回 true
    if (typeof window === 'undefined') return true;

    const requestOrigin = new URL(requestUrl).origin;
    return requestOrigin === window.location.origin;
  } catch {
    return false;  // 无效 URL
  }
}
```

## Cookie 值处理

中间件会自动处理 cookie 值：

1. **URL 解码**: cookie 值会 `decodeURIComponent` 后再添加到 header
2. **空值处理**: 空字符串 cookie → header 添加空字符串
3. **特殊字符**: 通过 regex 转义和解码处理

```ts
// 后端应期望 header 中是解码后的值
// Cookie: XSRF-TOKEN=test%20value
// Header: X-XSRF-TOKEN: test value
```
