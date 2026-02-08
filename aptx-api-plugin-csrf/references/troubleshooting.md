# Troubleshooting

## 问题 1: SSR/Node 报错 "document is not defined"

### 错误信息

```
Error: document is not defined
```

### 原因

在非浏览器环境使用默认的 `getCookie`，而默认实现依赖 `document.cookie`。

### 解决方案

**❌ 错误 - 在 SSR 中会崩溃**

```ts
const csrf = createCsrfMiddleware({});  // 默认使用 document.cookie
```

**✅ 正确 - 提供自定义 getCookie**

```ts
const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    // 你的自定义 cookie 读取逻辑
    return cookieStore.get(name)?.value;
  }
});
```

### Next.js 示例

```ts
import { cookies } from 'next/headers';

const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    const cookieStore = cookies();
    return cookieStore.get(name)?.value;
  }
});
```

### Express 示例

```ts
const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    const cookies = req.cookies;
    const value = cookies?.[name];
    return value ? decodeURIComponent(value) : undefined;
  }
});
```

---

## 问题 2: CSRF token 未出现在请求中

### 症状

```
Headers 中不包含 X-XSRF-TOKEN
```

### 可能原因

1. 浏览器中 cookie 不存在
2. `sameOriginOnly: true` + 跨域请求
3. cookie 名称与后端不匹配
4. `getCookie` 返回 undefined
5. cookie 名称拼写错误

### 调试步骤

#### 步骤 1: 检查 cookie 是否存在

```ts
const csrf = createCsrfMiddleware({
  cookieName: "XSRF-TOKEN",
  getCookie: (name) => {
    const value = /* 你的 cookie 逻辑 */;
    console.log(`[CSRF Debug] 查找 ${name}, 找到:`, value);
    return value;
  }
});
```

#### 步骤 2: 临时禁用 sameOriginOnly

```ts
const csrf = createCsrfMiddleware({
  sameOriginOnly: false,  // 临时禁用以调试
  getCookie: (name) => {
    const value = /* 你的 cookie 逻辑 */;
    console.log(`[CSRF Debug] 查找 ${name}, 找到:`, value);
    return value;
  }
});
```

如果禁用后 token 出现，说明是同源策略问题。

#### 步骤 3: 验证 cookie 名称

```ts
// 浏览器控制台
console.log(document.cookie.split(';').map(c => c.trim()));

// 或
console.log(document.cookie.includes('XSRF-TOKEN'));
```

#### 步骤 4: 检查跨域配置

```ts
// 测试当前请求是否被视为跨域
const testUrl = "https://api.example.com/data";
const isSameOrigin = new URL(testUrl).origin === window.location.origin;
console.log(`[CSRF Debug] 请求: ${testUrl}, 同源: ${isSameOrigin}`);
```

---

## 问题 3: 后端拒绝 CSRF token

### 症状

```
403 Forbidden: Invalid CSRF token
```

### 可能原因

1. 前后端 cookie 名称不匹配
2. header 名称不匹配
3. token 未正确 URL 解码
4. token 已过期
5. token 格式不正确

### 验证配置

#### 步骤 1: 确认名称匹配

```ts
// ✅ 确保名称与后端完全匹配
const csrf = createCsrfMiddleware({
  cookieName: "XSRF-TOKEN",        // 必须匹配后端 cookie 名称
  headerName: "X-XSRF-TOKEN",      // 必须匹配后端期望的 header
});
```

#### 步骤 2: 检查后端配置

```ts
// 后端（示例）
app.use(csrf({ cookie: { key: 'XSRF-TOKEN' } }));

app.use((req, res, next) => {
  const token = req.get('X-XSRF-TOKEN');
  // 验证 token...
});
```

#### 步骤 3: 验证 URL 解码

```ts
const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    const value = cookies.get(name);
    console.log(`[CSRF Debug] Cookie 原始值:`, value);

    // 中间件会自动 decode，后端应期望解码后的值
    // Cookie: XSRF-TOKEN=test%20value
    // Header: X-XSRF-TOKEN: test value
    return value;  // 不要手动 decode
  }
});
```

#### 步骤 4: 检查 token 过期

```ts
// 检查 cookie 的过期时间
document.cookie.split(';').forEach(c => {
  const [name, value] = c.split('=');
  if (name.trim() === 'XSRF-TOKEN') {
    console.log('Token 值:', value);
    // 检查后端是否验证 token 过期时间
  }
});
```

---

## 问题 4: 测试失败 "window is not defined"

### 错误信息

```
Error: window is not defined
```

### 原因

测试 `sameOriginOnly` 行为时没有 mock `window`。

### 解决方案

```ts
it("测试跨域行为", async () => {
  const originalWindow = (globalThis as any).window;
  (globalThis as any).window = {
    location: { origin: "https://example.com", href: "https://example.com" },
  };

  try {
    // 你的测试代码
    const res = await mw.handle(
      new Request({ method: "POST", url: "https://api.example.com/data" }),
      new Context({ id: "test-id", signal: new AbortController().signal }),
      async (req) => {
        expect(req.headers.get("X-XSRF-TOKEN")).toBeNull();
        return new Response({ /* ... */ });
      }
    );
  } finally {
    (globalThis as any).window = originalWindow;  // 始终在 finally 中恢复
  }
});
```

---

## 问题 5: Cookie 值被双重解码

### 症状

```
Header 中的 token 值不正确（例如空格变成 %2520）
```

### 原因

在 `getCookie` 中手动 `decodeURIComponent`，而中间件也会再次解码。

### 解决方案

**❌ 错误 - 双重解码**

```ts
getCookie: (name) => {
  const value = cookies.get(name);
  return value ? decodeURIComponent(value) : undefined;  // 会被中间件再次 decode！
}
```

**✅ 正确 - 返回原始值**

```ts
getCookie: (name) => {
  const value = cookies.get(name);
  return value;  // 中间件会自动 decode
}
```

**特殊情况 - 使用 js-cookie**

```ts
import Cookies from 'js-cookie';

getCookie: (name) => {
  return Cookies.get(name);  // js-cookie 已经 decode，中间件会再次 decode，需要调整
}
```

如果使用 js-cookie，可能需要修改中间件或使用自定义实现。

---

## 问题 6: 跨域请求需要 CSRF token

### 症状

跨域 API 请求返回 403，但同域请求正常。

### 原因

默认 `sameOriginOnly: true` 只在同源请求中添加 token。

### 解决方案

**⚠️ 谨慎使用 - 可能导致 token 泄露**

```ts
const csrf = createCsrfMiddleware({
  sameOriginOnly: false,  // 所有请求都添加 token
});
```

**更好的方案 - 使用 CORS**

```ts
// 1. 确保跨域域名受信任
// 2. 使用 CORS 白名单
// 3. 保持 sameOriginOnly: true

// 配置 CORS 允许的域名
app.use(cors({
  origin: ['https://api.example.com', 'https://trusted.com']
}));
```

---

## 问题 7: 请求 URL 无效导致 isSameOrigin 失败

### 症状

```
某些请求未添加 token，即使 cookie 存在
```

### 原因

无效的 URL 导致 `isSameOrigin` 返回 false。

### 调试

```ts
const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    return cookies.get(name);
  },
  // 添加日志查看 URL 解析
});

// 手动测试 URL
try {
  const url = new URL('/invalid');
  console.log('Origin:', url.origin);
} catch (e) {
  console.error('URL 无效:', e);
}
```

### 解决方案

确保所有请求 URL 都是有效的绝对或相对 URL。

---

## 问题 8: 浏览器中 cookie 被拒绝

### 症状

```
document.cookie 返回空字符串或无法读取
```

### 原因

1. Cookie 设置了 `HttpOnly` 标志
2. Cookie 的 `Domain` 不匹配
3. Cookie 的 `Path` 不匹配
4. Cookie 的 `SameSite` 策略阻止读取
5. HTTPS/HTTP 协议不匹配

### 调试

```ts
// 浏览器控制台
console.log(document.cookie);
console.log(document.cookie.includes('XSRF-TOKEN'));

// 检查 cookie 设置
console.log(navigator.cookieEnabled);
```

### 解决方案

#### 后端 Cookie 配置

```ts
// ✅ 正确 - 允许前端读取
res.cookie('XSRF-TOKEN', token, {
  httpOnly: false,  // 允许 JavaScript 读取
  secure: true,     // 仅 HTTPS
  sameSite: 'strict',
  domain: '.example.com',
  path: '/'
});
```

#### 检查 SameSite 策略

```ts
// strict: 最严格，仅同站请求发送
// lax: 允许顶级导航（推荐）
// none: 允许跨站（需要 secure）
res.cookie('XSRF-TOKEN', token, {
  sameSite: 'lax',  // 推荐设置
  secure: true      // sameSite='none' 时必须
});
```

---

## 调试最佳实践

### 添加条件日志

```ts
const csrf = createCsrfMiddleware({
  getCookie: (name) => {
    const value = cookies.get(name);

    if (process.env.NODE_ENV === 'development') {
      console.log(`[CSRF] 读取 ${name}:`, value);
      console.log(`[CSRF] URL:`, req?.url);
    }

    return value;
  }
});
```

### 检查请求头

```ts
const res = await client.fetch('/api/data', {
  method: 'POST'
});

console.log('Request headers:', res.request?.headers);
```

### 浏览器网络面板

1. 打开浏览器开发者工具
2. 切换到 Network 标签
3. 查看请求的 Headers
4. 确认 `X-XSRF-TOKEN` 是否存在

---

## 获取帮助

如果以上解决方案无法解决你的问题：

1. 检查中间件版本是否最新
2. 提供完整的错误信息和代码示例
3. 提供环境信息（浏览器、Node 版本、框架）
4. 提供最小可复现的代码示例
