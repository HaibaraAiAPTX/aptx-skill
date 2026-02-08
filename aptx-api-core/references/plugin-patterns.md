# Plugin Patterns

## 目录
- [Middleware vs Plugin](#middleware-vs-plugin)
- [创建 Plugin](#创建-plugin)
  - [Plugin 接口](#plugin-接口)
  - [替换 Transport](#替换-transport)
  - [监听事件](#监听事件)
  - [监听事件（返回清理函数）](#监听事件返回清理函数)
  - [替换 Decoder](#替换-decoder)
  - [替换 UrlResolver](#替换-urlresolver)
  - [替换 ErrorMapper](#替换-errormapper)
  - [组合多个能力](#组合多个能力)
  - [Plugin 注入 Middleware](#plugin-注入-middleware)
  - [链式应用 Plugins](#链式应用-plugins)
- [Plugin vs Middleware 选择指南](#plugin-vs-middleware-选择指南)
- [最佳实践](#最佳实践)

## Middleware vs Plugin

> Middleware 用于处理横切关注点（如日志、认证、重试），详见 [middleware-patterns.md](middleware-patterns.md)。Plugin 用于替换核心组件或监听事件。

| 能力 | Middleware | Plugin |
|------|-----------|--------|
| 修改请求/响应 | ✅ | ✅ (via use) |
| 替换 Transport | ❌ | ✅ |
| 替换 Decoder | ❌ | ✅ |
| 替换 UrlResolver | ❌ | ✅ |
| 替换 ErrorMapper | ❌ | ✅ |
| 监听事件 | ❌ | ✅ |
| 使用场景 | 横切关注点 | 替换核心组件 |

## 创建 Plugin

### Plugin 接口

```ts
import { Plugin, Registry } from "@aptx/api-core";

const myPlugin: Plugin = {
  setup(registry: Registry) {
    // 在这里配置插件
  }
};

client.apply(myPlugin);
```

### 替换 Transport

**何时使用**：从 Fetch 换到 Axios/XMLHttpRequest、WebSocket、其他 HTTP 客户端

```ts
import { Plugin, Registry, Transport, TransportResult } from "@aptx/api-core";
import axios from "axios";

const axiosTransportPlugin: Plugin = {
  setup(registry: Registry) {
    const axiosTransport: Transport = {
      async send(req, ctx) {
        const { data } = await axios({
          method: req.method,
          url: req.url,
          headers: Object.fromEntries(req.headers.entries()),
          data: req.body,
          timeout: req.timeoutMs,
          signal: ctx.signal,
        });

        return {
          status: data.status,
          headers: new Headers(data.headers),
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

### 监听事件

**何时使用**：请求日志、性能监控、错误追踪、A/B 测试

```ts
import { Plugin, Registry } from "@aptx/api-core";

const eventLoggingPlugin: Plugin = {
  setup(registry: Registry) {
    // 订阅事件
    registry.events.on("request:start", ({ req, ctx }) => {
      console.log(`[${ctx.id}] START ${req.method} ${req.url}`);
    });

    registry.events.on("request:end", ({ req, res, ctx, durationMs }) => {
      console.log(`[${ctx.id}] END ${res.status} ${req.url} (${durationMs}ms)`);
    });

    registry.events.on("request:error", ({ req, error, ctx, durationMs }) => {
      console.error(`[${ctx.id}] ERROR ${req.url} (${durationMs}ms)`, error);
    });

    registry.events.on("request:abort", ({ req, ctx, durationMs }) => {
      console.warn(`[${ctx.id}] ABORT ${req.url} (${durationMs}ms)`);
    });
  }
};

client.apply(eventLoggingPlugin);
```

### 监听事件（返回清理函数）

```ts
const eventTrackingPlugin: Plugin = {
  setup(registry: Registry) {
    const unsubscribe = registry.events.on("request:end", ({ req, res, ctx, durationMs }) => {
      // 发送到追踪服务
      trackRequest({
        method: req.method,
        url: req.url,
        status: res.status,
        duration: durationMs,
      });
    });

    // 返回的清理函数可以用于取消订阅
    return unsubscribe;
  }
};
```

### 替换 Decoder

**何时使用**：支持 Protobuf、MsgPack、自定义 JSON 解析、BigInt 支持

```ts
import { Plugin, Registry, ResponseDecoder, Response } from "@aptx/api-core";
import { MyMessage } from "./my-proto";

const protobufDecoderPlugin: Plugin = {
  setup(registry: Registry) {
    registry.setDecoder({
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
    });
  }
};

client.apply(protobufDecoderPlugin);
```

### 替换 UrlResolver

**何时使用**：复杂的 query 序列化、动态 baseURL 选择、URL 重写规则

```ts
import { Plugin, Registry, UrlResolver } from "@aptx/api-core";

const dynamicBaseURLPlugin = (getBaseURL: () => string): Plugin => ({
  setup(registry: Registry) {
    const dynamicUrlResolver: UrlResolver = {
      resolve(req, ctx) {
        const baseURL = getBaseURL();
        // 自定义 URL 解析逻辑
        return new URL(req.url, baseURL).toString();
      }
    };

    registry.setUrlResolver(dynamicUrlResolver);
  }
});

client.apply(dynamicBaseURLPlugin(() => window.location.origin));
```

### 替换 ErrorMapper

**何时使用**：根据业务状态码映射错误、添加额外的错误上下文、集成错误追踪服务

```ts
import { Plugin, Registry, ErrorMapper, HttpError } from "@aptx/api-core";

const businessErrorMapperPlugin: Plugin = {
  setup(registry: Registry) {
    const businessErrorMapper: ErrorMapper = {
      map(err, req, ctx, transport) {
        if (transport && transport.status === 403) {
          return new HttpError("权限不足", 403, req.url, transport.body, transport.headers, err);
        }

        if (transport && transport.status === 404) {
          return new HttpError("资源不存在", 404, req.url, transport.body, transport.headers, err);
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

### 组合多个能力

```ts
import { Plugin, Registry } from "@aptx/api-core";

const myPlugin: Plugin = {
  setup(registry: Registry) {
    // 1. 注册 middleware
    registry.use(loggingMiddleware);

    // 2. 替换 Transport
    registry.setTransport(customTransport);

    // 3. 监听事件
    registry.events.on("request:start", ({ req, ctx }) => {
      console.log(`[${ctx.id}] ${req.method} ${req.url}`);
    });
  }
};

client.apply(myPlugin);
```

### Plugin 注入 Middleware

> 详见 [middleware-patterns.md](middleware-patterns.md) 了解常见的 Middleware 实现模式，[context-bag.md](context-bag.md) 了解如何在 Middleware 间共享状态。

```ts
import { Plugin, Registry } from "@aptx/api-core";

const authPlugin: Plugin = ({
  store,
  refreshToken,
}: {
  store: TokenStore;
  refreshToken: () => Promise<string>;
}): Plugin => ({
  setup(registry: Registry) {
    const authMiddleware = createAuthMiddleware({
      store,
      refreshToken,
    });

    registry.use(authMiddleware);

    // 注册事件监听器
    registry.events.on("request:error", ({ error }) => {
      if (error instanceof HttpError && error.status === 401) {
        console.warn("Auth error:", error);
      }
    });
  }
});

client.apply(authPlugin({ store, refreshToken }));
```

### 链式应用 Plugins

```ts
client
  .apply(eventLoggingPlugin)
  .apply(customTransportPlugin)
  .apply(authPlugin({ store, refreshToken }));
```

## Plugin vs Middleware 选择指南

| 场景 | 选择 | 原因 |
|------|------|------|
| 添加请求头 | Middleware | 简单的请求/响应修改 |
| 日志记录 | Plugin | 需要监听事件 |
| 缓存 | Middleware | 横切关注点 |
| 替换 HTTP 客户端 | Plugin | 替换核心组件 |
| 添加认证 | Middleware | 横切关注点 |
| 支持 Protobuf | Plugin | 替换 Decoder |
| 错误追踪 | Plugin | 需要监听事件 |
| 请求重试 | Middleware | 横切关注点 |

## 最佳实践

1. ✅ Plugin 用于**替换核心组件**或**监听事件**
2. ✅ Middleware 用于**横切关注点**（日志、认证、重试等）
3. ✅ Plugin setup 中可以注册多个 middleware
4. ✅ Plugin 可以组合使用（`client.apply(p1).apply(p2)`）
5. ✅ 使用工厂函数创建带参数的 Plugin
6. ❌ 不要在 Plugin 中执行业务逻辑（应在 Middleware）
7. ❌ 不要在 Plugin 中修改 Request/Response（应在 Middleware）
