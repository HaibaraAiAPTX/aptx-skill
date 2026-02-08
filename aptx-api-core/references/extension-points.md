# Extension Points

## 目录
- [可扩展的组件](#可扩展的组件)
- [1. Transport - 底层传输层](#1-transport---底层传输层)
  - [接口定义](#接口定义)
  - [何时自定义](#何时自定义)
  - [示例：使用 Axios](#示例使用-axios)
  - [示例：使用 XMLHttpRequest](#示例使用-xmlhttprequest)
- [2. UrlResolver - URL 解析](#2-urlresolver---url-解析)
  - [接口定义](#接口定义-1)
  - [何时自定义](#何时自定义-1)
  - [示例：动态 baseURL](#示例动态-baseurl)
  - [示例：自定义 query 序列化](#示例自定义-query-序列化)
  - [示例：URL 重写](#示例url-重写)
- [3. BodySerializer - 请求体序列化](#3-bodyserializer---请求体序列化)
  - [接口定义](#接口定义-2)
  - [何时自定义](#何时自定义-2)
  - [示例：自定义日期序列化](#示例自定义日期序列化)
  - [示例：MsgPack 序列化](#示例msgpack-序列化)
- [4. ResponseDecoder - 响应解码](#4-responsedecoder---响应解码)
  - [接口定义](#接口定义-3)
  - [何时自定义](#何时自定义-3)
  - [示例：BigInt JSON 解析](#示例bigint-json-解析)
  - [示例：Protobuf 解码](#示例protobuf-解码)
  - [示例：统一错误响应格式](#示例统一错误响应格式)
- [5. ErrorMapper - 错误映射](#5-errormapper---错误映射)
  - [接口定义](#接口定义-4)
  - [何时自定义](#何时自定义-4)
  - [示例：业务错误映射](#示例业务错误映射)
  - [示例：错误追踪集成](#示例错误追踪集成)
- [选择扩展点指南](#选择扩展点指南)
- [组合多个扩展点](#组合多个扩展点)
- [最佳实践](#最佳实践-6)

`@aptx/api-core` 提供了 6 个核心扩展点，可以通过 Plugin 进行替换：

| 扩展点 | 接口 | 默认实现 | 用途 |
|--------|------|----------|------|
| Transport | `Transport` | `FetchTransport` | 底层 HTTP 传输 |
| UrlResolver | `UrlResolver` | `DefaultUrlResolver` | URL 解析和 query 序列化 |
| BodySerializer | `BodySerializer` | `DefaultBodySerializer` | 请求体序列化 |
| ResponseDecoder | `ResponseDecoder` | `DefaultResponseDecoder` | 响应解码 |
| ErrorMapper | `ErrorMapper` | `DefaultErrorMapper` | 错误映射 |

### 1. Transport - 底层传输层

**接口定义**：

```ts
import { Transport, TransportResult, Request, Context } from "@aptx/api-core";

interface Transport {
  send(req: Request, ctx: Context): Promise<TransportResult>;
}

interface TransportResult {
  status: number;
  headers: Headers;
  url: string;
  raw: unknown; // 原始响应对象
}
```

**何时自定义**：
- 从 Fetch 换到 Axios/XMLHttpRequest
- 添加底层代理、证书、重定向处理
- WebSocket 或其他非 HTTP 协议
- 需要特殊的 HTTP 客户端配置

**示例：使用 Axios**

```ts
import { Plugin, Registry, Transport, TransportResult, Request, Context } from "@aptx/api-core";
import axios, { AxiosRequestConfig } from "axios";

const axiosTransportPlugin: Plugin = {
  setup(registry: Registry) {
    const axiosTransport: Transport = {
      async send(req: Request, ctx: Context): Promise<TransportResult> {
        const config: AxiosRequestConfig = {
          method: req.method.toLowerCase() as any,
          url: req.url,
          headers: Object.fromEntries(req.headers.entries()),
          data: req.body,
          timeout: req.timeoutMs,
          signal: ctx.signal,
        };

        const { data, status, headers } = await axios(config);

        return {
          status,
          headers: new Headers(headers),
          url: req.url,
          raw: data,
        };
      }
    };

    registry.setTransport(axiosTransport);
  }
};

client.apply(axiosTransportPlugin);
```

**示例：使用 XMLHttpRequest**

```ts
const xhrTransportPlugin: Plugin = {
  setup(registry: Registry) {
    const xhrTransport: Transport = {
      async send(req, ctx) {
        return new Promise((resolve, reject) => {
          const xhr = new XMLHttpRequest();

          xhr.open(req.method, req.url);
          xhr.timeout = req.timeoutMs ?? 0;

          // 设置请求头
          req.headers.forEach((value, key) => {
            xhr.setRequestHeader(key, value);
          });

          xhr.onload = () => {
            resolve({
              status: xhr.status,
              headers: new Headers(xhr.getAllResponseHeaders().split("\r\n").map(h => {
                const [key, ...values] = h.split(": ");
                return [key, values.join(": ")] as [string, string];
              })),
              url: xhr.responseURL,
              raw: xhr.response,
            });
          };

          xhr.onerror = () => {
            reject(new Error("Network error"));
          };

          xhr.ontimeout = () => {
            reject(new Error("Request timeout"));
          };

          // 监听中止信号
          if (ctx.signal) {
            ctx.signal.addEventListener("abort", () => {
              xhr.abort();
              reject(new Error("Request aborted"));
            }, { once: true });
          }

          xhr.send(req.body as any);
        });
      }
    };

    registry.setTransport(xhrTransport);
  }
};

client.apply(xhrTransportPlugin);
```

---

### 2. UrlResolver - URL 解析

**接口定义**：

```ts
interface UrlResolver {
  resolve(req: Request, ctx: Context): string;
}
```

**何时自定义**：
- 复杂的 query 序列化逻辑
- 动态 baseURL 选择（多环境）
- URL 重写规则
- 自定义 query 参数格式

**示例：动态 baseURL**

```ts
import { Plugin, Registry, UrlResolver, Request, Context } from "@aptx/api-core";

const dynamicBaseURLPlugin = (getBaseURL: () => string): Plugin => ({
  setup(registry: Registry) {
    const dynamicUrlResolver: UrlResolver = {
      resolve(req: Request, ctx: Context): string {
        const baseURL = getBaseURL();

        try {
          const url = new URL(req.url, baseURL);

          // 添加默认 query 参数
          url.searchParams.set("timestamp", Date.now().toString());

          return url.toString();
        } catch {
          throw new Error(`Invalid URL: ${req.url}`);
        }
      }
    };

    registry.setUrlResolver(dynamicUrlResolver);
  }
});

// 使用示例
const baseURL = () => {
  if (window.location.hostname === "localhost") {
    return "http://localhost:3000";
  }
  return "https://api.example.com";
};

client.apply(dynamicBaseURLPlugin(baseURL));
```

**示例：自定义 query 序列化**

```ts
const customQuerySerializerPlugin: Plugin = {
  setup(registry: Registry) {
    const customUrlResolver: UrlResolver = {
      resolve(req, ctx) {
        const url = new URL(req.url, baseURL);

        // 自定义 query 序列化（数组参数格式）
        if (req.query && typeof req.query === "object") {
          Object.entries(req.query).forEach(([key, value]) => {
            if (Array.isArray(value)) {
              // 格式：tags[]=a&tags[]=b
              value.forEach(v => url.searchParams.append(`${key}[]`, String(v)));
            } else if (value !== undefined && value !== null) {
              url.searchParams.set(key, String(value));
            }
          });
        }

        return url.toString();
      }
    };

    registry.setUrlResolver(customUrlResolver);
  }
};
```

**示例：URL 重写**

```ts
const urlRewritePlugin = (rules: { pattern: RegExp; replacement: string }[]): Plugin => ({
  setup(registry: Registry) {
    const rewritingUrlResolver: UrlResolver = {
      resolve(req, ctx) {
        let url = req.url;

        // 应用重写规则
        for (const rule of rules) {
          url = url.replace(rule.pattern, rule.replacement);
        }

        return new URL(url, baseURL).toString();
      }
    };

    registry.setUrlResolver(rewritingUrlResolver);
  }
});

client.apply(urlRewritePlugin([
  { pattern: /^\/api\//, replacement: "/v1/api/" },
  { pattern: /^\/users\//, replacement: "/accounts/" },
]));
```

---

### 3. BodySerializer - 请求体序列化

**接口定义**：

```ts
interface BodySerializer {
  serialize(req: Request, ctx: Context): { body: any; headers?: HeadersInitLike };
}
```

**何时自定义**：
- 自定义 JSON 序列化（如日期格式）
- 支持 Protobuf、MsgPack 等格式
- 文件上传优化
- 自定义 Content-Type

**示例：自定义日期序列化**

```ts
import { Plugin, Registry, BodySerializer, Request, Context } from "@aptx/api-core";

const customDateSerializerPlugin: Plugin = {
  setup(registry: Registry) {
    const customBodySerializer: BodySerializer = {
      serialize(req: Request, ctx: Context) {
        if (req.body && typeof req.body === "object") {
          // 自定义序列化：日期转为 ISO 字符串
          const serialized = JSON.stringify(req.body, (key, value) => {
            if (value instanceof Date) {
              return value.toISOString();
            }
            return value;
          });

          return {
            body: serialized,
            headers: { "Content-Type": "application/json" },
          };
        }

        // fallback to default
        return new DefaultBodySerializer().serialize(req, ctx);
      }
    };

    registry.setBodySerializer(customBodySerializer);
  }
};

client.apply(customDateSerializerPlugin);
```

**示例：MsgPack 序列化**

```ts
import msgpack from "msgpack-lite";

const msgpackSerializerPlugin: Plugin = {
  setup(registry: Registry) {
    const msgpackBodySerializer: BodySerializer = {
      serialize(req, ctx) {
        if (req.meta?.bodyFormat === "msgpack" && req.body) {
          const encoded = msgpack.encode(req.body);
          return {
            body: encoded,
            headers: { "Content-Type": "application/msgpack" },
          };
        }

        // fallback to default
        return new DefaultBodySerializer().serialize(req, ctx);
      }
    };

    registry.setBodySerializer(msgpackBodySerializer);
  }
};

// 使用示例
await client.fetch("/api/data", {
  method: "POST",
  body: { foo: "bar" },
  meta: { bodyFormat: "msgpack" },
});
```

---

### 4. ResponseDecoder - 响应解码

**接口定义**：

```ts
interface ResponseDecoder {
  decode<T>(req: Request, transport: TransportResult, ctx: Context): Promise<Response<T>>;
}
```

**何时自定义**：
- 自定义 JSON 解析（如 BigInt 支持）
- Protobuf/MsgPack 解码
- 自定义错误处理逻辑
- 响应转换

**示例：BigInt JSON 解析**

```ts
import { Plugin, Registry, ResponseDecoder, Response, Request, Context } from "@aptx/api-core";

const bigintDecoderPlugin: Plugin = {
  setup(registry: Registry) {
    const bigintResponseDecoder: ResponseDecoder = {
      async decode<T>(req: Request, transport: TransportResult, ctx: Context): Promise<Response<T>> {
        const contentType = transport.headers.get("content-type");

        if (contentType?.includes("application/json")) {
          const text = await transport.raw.text();

          // 使用 BigInt 解析
          const data = JSON.parse(text, (key, value) => {
            if (typeof value === "string" && /^\d+$/.test(value)) {
              return BigInt(value);
            }
            return value;
          });

          return new Response({
            status: transport.status,
            headers: transport.headers,
            url: transport.url,
            data,
            raw: transport.raw,
          });
        }

        // fallback to default
        return new DefaultResponseDecoder().decode(req, transport, ctx);
      }
    };

    registry.setDecoder(bigintResponseDecoder);
  }
};

client.apply(bigintDecoderPlugin);
```

**示例：Protobuf 解码**

```ts
import { MyMessage } from "./my-proto";

const protobufDecoderPlugin: Plugin = {
  setup(registry: Registry) {
    const protobufResponseDecoder: ResponseDecoder = {
      async decode(req, transport, ctx) {
        if (req.meta?.responseType === "protobuf") {
          const buffer = await transport.raw.arrayBuffer();
          const message = MyMessage.decode(new Uint8Array(buffer));

          return new Response({
            status: transport.status,
            headers: transport.headers,
            url: transport.url,
            data: message,
            raw: transport.raw,
          });
        }

        // fallback to default
        return new DefaultResponseDecoder().decode(req, transport, ctx);
      }
    };

    registry.setDecoder(protobufResponseDecoder);
  }
};
```

**示例：统一错误响应格式**

```ts
interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details?: unknown;
  };
}

const errorFormatDecoderPlugin: Plugin = {
  setup(registry: Registry) {
    const errorFormatDecoder: ResponseDecoder = {
      async decode(req, transport, ctx) {
        const res = await new DefaultResponseDecoder().decode(req, transport, ctx);

        // 如果是错误响应，转换格式
        if (transport.status >= 400) {
          const data = res.data as ErrorResponse;

          if (data?.error) {
            throw new HttpError(
              data.error.message,
              transport.status,
              transport.url,
              data.error.details,
              transport.headers
            );
          }
        }

        return res;
      }
    };

    registry.setDecoder(errorFormatDecoder);
  }
};
```

---

### 5. ErrorMapper - 错误映射

**接口定义**：

```ts
interface ErrorMapper {
  map(err: unknown, req: Request, ctx: Context, transport?: TransportResult): Error;
}
```

**何时自定义**：
- 根据业务状态码映射错误
- 添加额外的错误上下文
- 集成错误追踪服务
- 自定义错误类型

**示例：业务错误映射**

```ts
import { Plugin, Registry, ErrorMapper, HttpError, ConfigError } from "@aptx/api-core";

const businessErrorMapperPlugin: Plugin = {
  setup(registry: Registry) {
    const businessErrorMapper: ErrorMapper = {
      map(err, req, ctx, transport) {
        // 如果已经是 UniReqError，直接返回
        if (err instanceof UniReqError) return err;

        // 根据 HTTP 状态码映射业务错误
        if (transport) {
          switch (transport.status) {
            case 401:
              return new HttpError("未授权，请登录", 401, req.url, undefined, transport.headers, err);
            case 403:
              return new HttpError("权限不足", 403, req.url, undefined, transport.headers, err);
            case 404:
              return new HttpError("资源不存在", 404, req.url, undefined, transport.headers, err);
            case 422:
              return new HttpError("请求参数错误", 422, req.url, undefined, transport.headers, err);
            case 500:
              return new HttpError("服务器内部错误", 500, req.url, undefined, transport.headers, err);
          }
        }

        // 从响应体解析业务错误代码
        if (transport?.raw && typeof transport.raw === "object") {
          const body = transport.raw as { error?: { code?: string; message?: string } };
          if (body?.error?.code) {
            return new HttpError(
              body.error.message || `业务错误: ${body.error.code}`,
              transport.status,
              req.url,
              body.error,
              transport.headers,
              err
            );
          }
        }

        // fallback to default
        return new DefaultErrorMapper().map(err, req, ctx, transport);
      }
    };

    registry.setErrorMapper(businessErrorMapper);
  }
};

client.apply(businessErrorMapperPlugin);
```

**示例：错误追踪集成**

```ts
const errorTrackingMapperPlugin: Plugin = {
  setup(registry: Registry) {
    const errorTrackingMapper: ErrorMapper = {
      map(err, req, ctx, transport) {
        // 先使用默认映射
        const mappedError = new DefaultErrorMapper().map(err, req, ctx, transport);

        // 发送到错误追踪服务
        if (process.env.NODE_ENV === "production") {
          trackError(mappedError, {
            url: req.url,
            method: req.method,
            status: transport?.status,
            requestId: ctx.id,
          });
        }

        return mappedError;
      }
    };

    registry.setErrorMapper(errorTrackingMapper);
  }
};
```

---

## 选择扩展点指南

> 详见 [testing-guide.md](testing-guide.md) 了解如何测试自定义扩展点。

| 需求 | 推荐方式 | 原因 |
|------|----------|------|
| 修改请求头/添加参数 | Middleware | 最简单、最常见的场景 |
| 修改响应数据 | Middleware | 不影响核心功能 |
| 记录请求/响应日志 | Plugin（事件） | 需要监听事件 |
| 替换 HTTP 客户端 | Plugin（Transport） | 需要替换底层实现 |
| 修改 URL 解析逻辑 | Plugin（UrlResolver） | 需要替换 URL 解析 |
| 自定义序列化格式 | Plugin（BodySerializer） | 需要替换序列化逻辑 |
| 自定义解码格式 | Plugin（Decoder） | 需要替换解码逻辑 |
| 自定义错误类型 | Plugin（ErrorMapper） | 需要替换错误映射 |
| 添加认证 | Middleware | 横切关注点 |
| 添加重试 | Middleware | 横切关注点 |
| 添加缓存 | Middleware | 横切关注点 |

## 组合多个扩展点

> 详见 [plugin-patterns.md](plugin-patterns.md) 了解 Plugin 的完整使用模式，包括如何注册 Middleware、监听事件等。

```ts
const myCompletePlugin: Plugin = {
  setup(registry: Registry) {
    // 1. 替换 Transport
    registry.setTransport(customTransport);

    // 2. 替换 UrlResolver
    registry.setUrlResolver(customUrlResolver);

    // 3. 替换 Decoder
    registry.setDecoder(customDecoder);

    // 4. 替换 ErrorMapper
    registry.setErrorMapper(customErrorMapper);

    // 5. 注册 Middleware
    registry.use(authMiddleware);
    registry.use(loggingMiddleware);

    // 6. 监听事件
    registry.events.on("request:end", ({ req, res, ctx }) => {
      console.log(`[${ctx.id}] ${res.status} ${req.url}`);
    });
  }
};

client.apply(myCompletePlugin);
```

## 最佳实践

1. ✅ 优先使用 Middleware（最简单、最常见）
2. ✅ 只有 Middleware 无法满足时才替换组件
3. ✅ 替换组件时保持接口一致性
4. ✅ 在 Plugin 中组合多个扩展点
5. ✅ 使用工厂函数创建带参数的 Plugin
6. ❌ 不要在 Plugin 中执行业务逻辑（应在 Middleware）
7. ❌ 不要在替换组件时丢失默认行为（应 fallback）
8. ❌ 不要假设组件的调用顺序
