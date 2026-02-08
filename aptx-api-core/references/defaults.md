# 默认实现

`@aptx/api-core` 提供了以下默认实现，可以通过 RequestClient 构造函数或 Plugin 替换。

## 目录
- [DefaultUrlResolver](#defaulturlresolver)
  - [功能](#功能)
  - [默认行为](#默认行为)
  - [错误处理](#错误处理)
  - [自定义 query 序列化](#自定义-query-序列化)
  - [最佳实践](#最佳实践-1)
- [DefaultBodySerializer](#defaultbodyserializer)
  - [默认行为](#默认行为-1)
  - [Content-Type 自动设置](#content-type-自动设置)
  - [最佳实践](#最佳实践-2)
- [DefaultResponseDecoder](#defaultresponsedecoder)
  - [响应类型选择优先级](#响应类型选择优先级)
  - [自动类型检测](#自动类型检测)
  - [strictDecode 严格模式](#strictdecode-严格模式)
  - [错误处理](#错误处理)
  - [支持的响应类型](#支持的响应类型)
  - [最佳实践](#最佳实践-3)
- [DefaultErrorMapper](#defaulterrormapper)
  - [错误映射规则](#错误映射规则)
  - [错误类型](#错误类型)
  - [TIMEOUT_BAG_KEY 的作用](#timeout_bag_key-的作用)
  - [自定义错误映射](#自定义错误映射)
  - [最佳实践](#最佳实践-4)
- [FetchTransport](#fetchtransport)
  - [功能](#功能-1)
  - [默认行为](#默认行为-2)
  - [Progress 回调（best-effort）](#progress-回调best-effort)
- [SimpleEventBus](#simpleeventbus)
  - [功能](#功能-2)
  - [使用示例](#使用示例)
  - [支持的事件](#支持的事件)
- [替换默认实现](#替换默认实现)
  - [在构造函数中替换](#在构造函数中替换)
  - [通过 Plugin 替换](#通过-plugin-替换)
  - [最佳实践](#最佳实践-5)

## DefaultUrlResolver

默认 URL 解析器，处理 baseURL 和查询参数。

### 功能

1. **baseURL 解析**：将相对 URL 与 baseURL 合并
2. **query 序列化**：自动序列化查询参数
3. **类型支持**：支持 Record、URLSearchParams、Array 格式的 query

### 默认行为

```ts
const resolver = new DefaultUrlResolver("https://api.example.com");

// 相对 URL
resolver.resolve({ url: "/user" }, ctx);
// → "https://api.example.com/user"

// 带 query
resolver.resolve({ url: "/user", query: { id: 123 } }, ctx);
// → "https://api.example.com/user?id=123"

// 数组 query
resolver.resolve({ url: "/user", query: { tags: ["a", "b"] } }, ctx);
// → "https://api.example.com/user?tags=a&tags=b"

// URLSearchParams
resolver.resolve({ url: "/user", query: new URLSearchParams("id=123") }, ctx);
// → "https://api.example.com/user?id=123"
```

### 错误处理

```ts
// 没有 baseURL + 相对 URL → ConfigError
const resolver = new DefaultUrlResolver();
resolver.resolve({ url: "/user" }, ctx);
// → throws ConfigError("Relative URL is not allowed without baseURL")

// 绝对 URL 不需要 baseURL
resolver.resolve({ url: "https://example.com/user" }, ctx);
// → OK
```

### 自定义 query 序列化

```ts
const customQuerySerializer = (query, url) => {
  // 自定义序列化逻辑
  return `${url}?custom=${encodeURIComponent(JSON.stringify(query))}`;
};

const resolver = new DefaultUrlResolver(
  "https://api.example.com",
  customQuerySerializer
);
```

### 最佳实践

1. ✅ 生产环境始终提供 baseURL
2. ✅ 复杂 query 序列化使用自定义序列化器
3. ❌ 避免传递 `undefined` 或 `null` 作为 query 值（会被忽略）

---

## DefaultBodySerializer

默认请求体序列化器，自动处理 JSON 和 FormData。

### 默认行为

```ts
const serializer = new DefaultBodySerializer();

// JSON 对象
serializer.serialize({
  url: "/user",
  method: "POST",
  body: { name: "test" }
}, ctx);
// → { body: '{"name":"test"}', headers: { "Content-Type": "application/json" } }

// FormData
serializer.serialize({
  url: "/upload",
  method: "POST",
  body: new FormData()
}, ctx);
// → { body: FormData, headers: {} }（Content-Type 由浏览器自动设置）

// string
serializer.serialize({
  url: "/text",
  method: "POST",
  body: "hello"
}, ctx);
// → { body: "hello", headers: { "Content-Type": "text/plain;charset=UTF-8" } }

// null/undefined
serializer.serialize({
  url: "/no-body",
  method: "POST",
  body: null
}, ctx);
// → { body: null, headers: {} }
```

### Content-Type 自动设置

| body 类型 | Content-Type |
|-----------|--------------|
| object | `application/json` |
| FormData | 不设置（浏览器自动设置） |
| string | `text/plain;charset=UTF-8` |
| Blob | `blob.type` |
| ArrayBuffer/ArrayBufferView | `application/octet-stream` |
| URLSearchParams | `application/x-www-form-urlencoded` |

### 最佳实践

1. ✅ 使用 FormData 上传文件
2. ✅ JSON 请求自动序列化，无需手动 JSON.stringify
3. ❌ 不要手动设置 Content-Type（会被自动覆盖）

---

## DefaultResponseDecoder

默认响应解码器，根据 Content-Type 自动选择解析方式。

### 响应类型选择优先级

1. `req.meta.responseType`（显式指定）
2. `Content-Type` 自动检测（json/text）
3. `defaultResponseType`（默认配置）
4. `raw`（兜底）

### 自动类型检测

```ts
const decoder = new DefaultResponseDecoder();

// JSON 响应
decoder.decode(
  { meta: {} },
  {
    status: 200,
    headers: new Headers({ "content-type": "application/json" }),
    raw: new Response('{"ok":true}'),
    url: "https://example.com"
  },
  ctx
);
// → Response<{ data: { ok: true }, status: 200, ... }>

// 文本响应
decoder.decode(
  { meta: {} },
  {
    status: 200,
    headers: new Headers({ "content-type": "text/plain" }),
    raw: new Response("hello"),
    url: "https://example.com"
  },
  ctx
);
// → Response<{ data: "hello", status: 200, ... }>

// Blob 响应
decoder.decode(
  { meta: { responseType: "blob" } },
  {
    status: 200,
    headers: new Headers(),
    raw: new Response(new Blob(["data"])),
    url: "https://example.com"
  },
  ctx
);
// → Response<{ data: Blob, status: 200, ... }>
```

### strictDecode 严格模式

启用严格模式，无法确定类型时抛出错误：

```ts
const decoder = new DefaultResponseDecoder({ strictDecode: true });

// 没有 Content-Type 且未指定 responseType
decoder.decode(
  { meta: {} },
  {
    status: 200,
    headers: new Headers(), // 没有 Content-Type
    raw: new Response("data"),
    url: "https://example.com"
  },
  ctx
);
// → throws Error("Unable to determine response type")
```

### 错误处理

HTTP 错误状态码（< 200 或 >= 300）自动抛出 `HttpError`：

```ts
decoder.decode(
  { meta: {} },
  {
    status: 404,
    headers: new Headers({ "content-type": "application/json" }),
    raw: new Response('{"error":"not found"}'),
    url: "https://example.com"
  },
  ctx
);
// → throws HttpError("HTTP 404", 404, "https://example.com", { error: "not found" })
```

### 支持的响应类型

| responseType | 解析方式 | 用途 |
|-------------|---------|------|
| `"json"` | `raw.json()` | API 响应 |
| `"text"` | `raw.text()` | 文本内容 |
| `"blob"` | `raw.blob()` | 文件下载 |
| `"arrayBuffer"` | `raw.arrayBuffer()` | 二进制数据 |
| `"raw"` | 不解析 | 原始 Response 对象 |

### 最佳实践

1. ✅ API 响应默认使用 JSON（自动检测）
2. ✅ 文件下载使用 `responseType: "blob"`
3. ✅ 复杂场景使用 strictDecode 避免意外行为
4. ❌ 不要忽略 4xx/5xx 错误（会自动抛出 HttpError）

---

## DefaultErrorMapper

默认错误映射器，将底层错误转换为统一错误类型。

### 错误映射规则

```ts
const mapper = new DefaultErrorMapper();

// UniReqError → 直接返回
mapper.map(new HttpError("404", 404, "url"), req, ctx);
// → HttpError("404", 404, "url")

// TIMEOUT_BAG_KEY 设置 → TimeoutError
ctx.bag.set(TIMEOUT_BAG_KEY, true);
mapper.map(new Error("timeout"), req, ctx);
// → TimeoutError("Request timed out", Error)

// AbortError → CanceledError
mapper.map({ name: "AbortError" }, req, ctx);
// → CanceledError("Request canceled", { name: "AbortError" })

// 其他 → NetworkError
mapper.map(new Error("network"), req, ctx);
// → NetworkError("Network error", Error)
```

### 错误类型

| 错误类型 | 条件 | 用途 |
|---------|------|------|
| `TimeoutError` | `ctx.bag.get(TIMEOUT_BAG_KEY) === true` | 请求超时 |
| `CanceledError` | 原始错误 `name === "AbortError"` | 用户取消 |
| `NetworkError` | 其他所有错误 | 网络错误 |
| `UniReqError` 子类 | 已经是 UniReqError | 保持原样 |

### TIMEOUT_BAG_KEY 的作用

区分超时和用户取消：

```ts
// 超时
ctx.bag.set(TIMEOUT_BAG_KEY, true);
// → TimeoutError

// 用户取消
// → CanceledError
```

### 自定义错误映射

```ts
import { DefaultErrorMapper } from "@aptx/api-core";

class CustomErrorMapper extends DefaultErrorMapper {
  map(err, req, ctx, transport) {
    const baseError = super.map(err, req, ctx, transport);

    // 添加额外的上下文
    if (transport?.status === 401) {
      baseError.message = "未授权，请重新登录";
    }

    return baseError;
  }
}
```

### 最佳实践

1. ✅ 自定义错误映射继承 DefaultErrorMapper
2. ✅ 根据 HTTP 状态码添加业务友好的错误消息
3. ✅ 保留原始 cause 信息用于调试
4. ❌ 不要吞掉错误信息

---

## FetchTransport

默认基于 Fetch API 的传输层。

### 功能

1. 发送 HTTP 请求
2. 支持 AbortSignal
3. 支持 timeout（通过 AbortSignal）
4. 支持 progress（best-effort）

### 默认行为

```ts
const transport = new FetchTransport(serializer);

await transport.send({
  method: "POST",
  url: "https://example.com",
  headers: new Headers({ "Content-Type": "application/json" }),
  body: '{"key":"value"}',
}, ctx);
// → TransportResult { status, headers, url, raw: Response }
```

### Progress 回调（best-effort）

需要 RequestMeta 中提供进度回调：

```ts
const req = new Request({
  method: "POST",
  url: "/upload",
  body: file,
  meta: {
    onUploadProgress: ({ loaded, total, progress }) => {
      console.log(`Upload: ${(progress! * 100).toFixed(1)}%`);
    },
    onDownloadProgress: ({ loaded, total, progress }) => {
      console.log(`Download: ${(progress! * 100).toFixed(1)}%`);
    },
  },
});
```

**约束**：
- 仅在 FetchTransport 中生效
- 需要服务端返回 `Content-Length` 头
- `total` 可能是 `undefined`

---

## SimpleEventBus

默认事件总线实现。

### 功能

1. 订阅事件：`events.on(eventName, handler)`
2. 触发事件：`events.emit(eventName, payload)`
3. 取消订阅：返回清理函数

### 使用示例

```ts
const bus = new SimpleEventBus();

// 订阅事件
const unsubscribe = bus.on("request:start", ({ req, ctx }) => {
  console.log(`[${ctx.id}] ${req.method} ${req.url}`);
});

// 触发事件
bus.emit("request:start", { req, ctx });

// 取消订阅
unsubscribe();
```

### 支持的事件

| 事件名 | Payload | 触发时机 |
|--------|---------|----------|
| `request:start` | `{ req, ctx }` | 请求开始 |
| `request:end` | `{ req, res, ctx, durationMs, attempt }` | 请求成功 |
| `request:error` | `{ req, error, ctx, durationMs, attempt }` | 请求失败（不包括 abort） |
| `request:abort` | `{ req, ctx, durationMs }` | 请求被中止 |

---

## 替换默认实现

### 在构造函数中替换

```ts
const client = new RequestClient({
  transport: customTransport,
  urlResolver: customUrlResolver,
  bodySerializer: customBodySerializer,
  decoder: customDecoder,
  errorMapper: customErrorMapper,
});
```

### 通过 Plugin 替换

```ts
const plugin: Plugin = {
  setup(registry) {
    registry.setTransport(customTransport);
    registry.setUrlResolver(customUrlResolver);
    registry.setBodySerializer(customBodySerializer);
    registry.setDecoder(customDecoder);
    registry.setErrorMapper(customErrorMapper);
  }
};

client.apply(plugin);
```

### 最佳实践

1. ✅ 优先使用默认实现（满足大多数场景）
2. ✅ 只有确实需要时才替换
3. ✅ 替换时保持接口一致性
4. ❌ 不要替换核心实现，除非有明确需求
