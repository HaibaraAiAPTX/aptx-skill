# Environment Adaptation

## 浏览器环境

浏览器环境使用默认实现，无需额外配置：

```ts
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";

const csrf = createCsrfMiddleware({
  cookieName: "XSRF-TOKEN",
  headerName: "X-XSRF-TOKEN",
  sameOriginOnly: true,
  // 浏览器环境下不需要 getCookie，自动使用 document.cookie
});
```

### 默认行为

- `getCookie`: 自动使用 `document.cookie`
- `isSameOrigin`: 自动检查 `window.location.origin`

## Express/Node.js 环境

Node.js 环境需要提供自定义 `getCookie`：

```ts
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";

// Express 风格的 req/res
const csrf = createCsrfMiddleware({
  cookieName: "XSRF-TOKEN",
  headerName: "X-XSRF-TOKEN",
  sameOriginOnly: true,
  getCookie: (name) => {
    // 从你的 cookie store 读取（如 cookie-parser）
    const cookies = req.cookies; // 或 req.headers.cookie
    const value = cookies?.[name];
    return value ? decodeURIComponent(value) : undefined;
  }
});
```

### cookie-parser 集成

```ts
import cookieParser from 'cookie-parser';

app.use(cookieParser());

const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    return req.cookies?.[name]; // cookie-parser 解析后的对象
  }
});
```

### 手动解析 cookie header

```ts
const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    const cookieHeader = req.headers.cookie || '';
    const pattern = new RegExp(`(?:^|; )${name.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")}=([^;]*)`);
    const match = cookieHeader.match(pattern);
    const value = match?.[1];
    return value ? decodeURIComponent(value) : undefined;
  }
});
```

## Next.js SSR 环境

Next.js 使用 `cookies()` API：

```ts
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";
import { cookies } from 'next/headers';

const csrf = createCsrfMiddleware({
  cookieName: "XSRF-TOKEN",
  headerName: "X-XSRF-TOKEN",
  sameOriginOnly: true,
  getCookie: (name) => {
    const cookieStore = cookies();
    return cookieStore.get(name)?.value;
  }
});
```

### App Router (Server Components)

```ts
// app/page.tsx
import { cookies } from 'next/headers';
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";

export default async function Page() {
  const csrf = createCsrfMiddleware({
    getCookie: (name) => {
      const cookieStore = cookies();
      return cookieStore.get(name)?.value;
    }
  });

  // 使用中间件...
}
```

### App Router (Route Handlers)

```ts
// app/api/handlers/route.ts
import { cookies } from 'next/headers';
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";

export async function POST(req: Request) {
  const csrf = createCsrfMiddleware({
    getCookie: (name) => {
      const cookieStore = cookies();
      return cookieStore.get(name)?.value;
    }
  });

  // 处理请求...
}
```

### Pages Router (getServerSideProps)

```ts
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";

export async function getServerSideProps(context) {
  const { req } = context;

  const csrf = createCsrfMiddleware({
    getCookie: (name) => {
      const cookieHeader = req.headers.cookie || '';
      // 手动解析
      const pattern = new RegExp(`(?:^|; )${name.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")}=([^;]*)`);
      const match = cookieHeader.match(pattern);
      return match?.[1];
    }
  });

  return { props: {} };
}
```

## 自定义 Cookie Store

对于使用自定义 cookie 管理的场景：

```ts
import { createCsrfMiddleware } from "@aptx/api-plugin-csrf";

const csrf = createCsrfMiddleware({
  cookieName: "XSRF-TOKEN",
  headerName: "X-XSRF-TOKEN",
  sameOriginOnly: true,
  getCookie: (name) => {
    // 从你的自定义 cookie store 读取
    return myCookieStore.get(name);
  }
});
```

### 示例：使用 js-cookie

```ts
import Cookies from 'js-cookie';

const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    return Cookies.get(name); // js-cookie 自动处理编码/解码
  }
});
```

### 示例：使用 localStorage

```ts
const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    return localStorage.getItem(name) || undefined;
  }
});
```

## 环境差异总结

| 环境 | `document.cookie` | `window.location` | 默认 `getCookie` | 默认 `isSameOrigin` |
|------|-------------------|-------------------|-------------------|---------------------|
| 浏览器 | ✓ 可用 | ✓ 可用 | ✓ 使用 document.cookie | ✓ 检查 origin |
| SSR (Next.js) | ✗ undefined | ✗ undefined | ✗ 需要自定义 getCookie | ✓ 返回 true（始终同源） |
| Node.js | ✗ undefined | ✗ undefined | ✗ 需要自定义 getCookie | ✓ 返回 true（始终同源） |

## isSameOrigin 在 SSR 的行为

在 SSR/Node 环境中：

```ts
function isSameOrigin(requestUrl: string): boolean {
  // SSR 环境没有 window，默认返回 true
  if (typeof window === 'undefined') return true;
  // ...
}
```

**原因:** SSR 场景下请求通常来自服务器内部，视为同源。

**覆盖:** 如需自定义 SSR 同源逻辑：

```ts
// 需要修改中间件源码或使用自定义实现
```

## Cookie 值处理细节

### URL 解码

中间件自动对 cookie 值进行 `decodeURIComponent`：

```ts
// Cookie: XSRF-TOKEN=test%20value%26symbol
// Header: X-XSRF-TOKEN: test value&symbol
```

**自定义 getCookie 时需要注意：**

```ts
// ✅ 正确：返回原始 cookie 值（可能已编码）
getCookie: (name) => {
  return cookies.get(name);  // 不要手动 decode
}

// ❌ 错误：手动 decode 会导致双重解码
getCookie: (name) => {
  const value = cookies.get(name);
  return value ? decodeURIComponent(value) : undefined;  // 会被中间件再次 decode！
}
```

### 特殊字符

中间件通过 regex 转义处理特殊字符：

```ts
const pattern = new RegExp(`(?:^|; )${name.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")}=([^;]*)`);
```

支持 cookie 名称中的特殊字符，如 `.`、`-`、`_`。
