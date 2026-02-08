# 常见问题

## 目录
- [Q: Request headers 没有被修改？](#q-request-headers-没有被修改)
- [Q: Progress 回调没有触发？](#q-progress-回调没有触发)
- [Q: 如何区分超时和用户取消？](#q-如何区分超时和用户取消)
- [Q: 如何防止 auth middleware 无限重试？](#q-如何防止-auth-middleware-无限重试)
- [Q: 何时使用 Middleware vs Plugin？](#q-何时使用-middleware-vs-plugin)
- [Q: 如何在 middleware 间共享数据？](#q-如何在-middleware-间共享数据)
- [Q: 如何测试自定义 middleware？](#q-如何测试自定义-middleware)
- [Q: Response 数据类型不对怎么办？](#q-response-数据类型不对怎么办)
- [Q: 如何自定义 JSON 序列化？](#q-如何自定义-json-序列化)

---

### Q: Request headers 没有被修改？

**A**: Request 是不可变的，必须使用 `req.with()` 创建新实例。

```ts
// ❌ 错误：直接修改无效
req.headers.set("X-Custom", "value");

// ✅ 正确：使用 req.with()
const modifiedReq = req.with({
  headers: { "X-Custom": "value" }
});
```

详见：[middleware-patterns.md - 最佳实践](references/middleware-patterns.md#最佳实践)

---

### Q: Progress 回调没有触发？

**A**: Progress 回调仅在以下条件下生效：
1. 使用 `FetchTransport`（默认 transport）
2. 服务端返回 `Content-Length` 头
3. 在 `RequestMeta` 中设置回调

```ts
await client.fetch("/upload", {
  method: "POST",
  body: largeFile,
  meta: {
    onUploadProgress: ({ loaded, total, progress }) => {
      console.log(`Upload: ${(progress! * 100).toFixed(1)}%`);
    },
  },
});
```

**注意**: 这是 best-effort 功能，不保证精确性。

详见：[SKILL.md - 请求元数据 - 进度回调](../SKILL.md#进度回调best-effort)

---

### Q: 如何区分超时和用户取消？

**A**: 检查 `ctx.bag.get(TIMEOUT_BAG_KEY)`。

```ts
import { TIMEOUT_BAG_KEY } from "@aptx/api-core";

const timeoutCheckingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    try {
      return await next(req, ctx);
    } catch (err) {
      if (ctx.bag.get(TIMEOUT_BAG_KEY) === true) {
        console.log("Request timed out");
      } else {
        console.log("Request was cancelled by user");
      }
      throw err;
    }
  }
};
```

详见：[context-bag.md - TIMEOUT_BAG_KEY](references/context-bag.md#timeout_bag_key---超时检测)

---

### Q: 如何防止 auth middleware 无限重试？

**A**: 使用 Context Bag 存储重试计数。

```ts
import { createBagKey } from "@aptx/api-core";

const AUTH_RETRY_KEY = createBagKey("auth_retry");

const authMiddleware: Middleware = {
  async handle(req, ctx, next) {
    try {
      return await next(req, ctx);
    } catch (err) {
      const error = err as Error;
      const shouldRefresh = error instanceof HttpError && error.status === 401;

      if (!shouldRefresh) throw error;

      const currentRetry = (ctx.bag.get(AUTH_RETRY_KEY) as number) ?? 0;
      if (currentRetry >= 1) throw error; // 防止无限重试

      ctx.bag.set(AUTH_RETRY_KEY, currentRetry + 1);

      // 刷新 token 并重试
      const newToken = await refreshToken();
      const retryReq = req.with({
        headers: { "Authorization": `Bearer ${newToken}` }
      });
      return await next(retryReq, ctx);
    }
  }
};
```

详见：[context-bag.md - 防止无限循环](references/context-bag.md#2-防止无限循环参考-auth-middleware)

---

### Q: 何时使用 Middleware vs Plugin？

**A**: 简单规则：

| 需求 | 选择 | 原因 |
|------|------|------|
| 修改请求头/添加参数 | Middleware | 简单的请求/响应修改 |
| 记录请求/响应日志 | Plugin（事件） | 需要监听事件 |
| 缓存 | Middleware | 横切关注点 |
| 替换 HTTP 客户端 | Plugin（Transport） | 需要替换底层实现 |
| 修改 URL 解析逻辑 | Plugin（UrlResolver） | 需要替换 URL 解析 |
| 自定义序列化格式 | Plugin（BodySerializer） | 需要替换序列化逻辑 |
| 自定义解码格式 | Plugin（Decoder） | 需要替换解码逻辑 |
| 添加认证 | Middleware | 横切关注点 |
| 添加重试 | Middleware | 横切关注点 |

详见：[plugin-patterns.md - Plugin vs Middleware 选择指南](references/plugin-patterns.md#plugin-vs-middleware-选择指南)

---

### Q: 如何在 middleware 间共享数据？

**A**: 使用 `ctx.bag` 和 `createBagKey()`。

```ts
import { createBagKey } from "@aptx/api-core";

const USER_KEY = createBagKey("user");

// 设置数据
const userFetchingMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const user = await fetchUser(req.headers.get("X-User-ID"));
    ctx.bag.set(USER_KEY, user);
    return next(req, ctx);
  }
};

// 读取数据
const authMiddleware: Middleware = {
  async handle(req, ctx, next) {
    const user = ctx.bag.get(USER_KEY);
    if (!user || !user.isAdmin && req.url.startsWith("/admin")) {
      throw new Error("Permission denied");
    }
    return next(req, ctx);
  }
};
```

**重要**:
- 始终使用 `createBagKey()` 创建 key
- Bag 数据是请求作用域的（每个请求独立）
- 使用 TypeScript 类型断言：`ctx.bag.get(KEY) as Type`

详见：[context-bag.md - 使用示例](references/context-bag.md#使用示例)

---

### Q: 如何测试自定义 middleware？

**A**: 使用 mock `next()` 函数。

```ts
describe("myMiddleware", () => {
  it("should add custom header", async () => {
    const mw: Middleware = {
      async handle(req, ctx, next) {
        const modifiedReq = req.with({
          headers: { "X-Custom": "value" }
        });
        return next(modifiedReq, ctx);
      }
    };

    const req = new Request({
      method: "GET",
      url: "https://example.com",
      headers: { "X-Base": "1" }
    });

    const ctx = new Context({
      id: "test-1",
      signal: new AbortController().signal
    });

    const next = async (r: Request, c: Context) => {
      return new Response({
        status: 200,
        headers: new Headers(),
        url: r.url,
        data: "ok",
        raw: {},
      });
    };

    const res = await mw.handle(req, ctx, next);

    expect(req.headers.get("X-Custom")).toBe("value");
    expect(res.data).toBe("ok");
  });
});
```

详见：[testing-guide.md - 测试 Middleware](references/testing-guide.md#测试-middleware)

---

### Q: Response 数据类型不对怎么办？

**A**: 显式指定 `responseType`。

```ts
// JSON 响应（默认）
const jsonRes = await client.fetch("/api/data");
// → jsonRes.data: T

// 文本响应
const textRes = await client.fetch("/data", {
  meta: { responseType: "text" },
});
// → textRes.data: string

// Blob 响应（文件下载）
const blobRes = await client.fetch("/file.pdf", {
  meta: { responseType: "blob" },
});
// → blobRes.data: Blob
```

详见：[SKILL.md - 请求元数据 - 显式响应类型](../SKILL.md#显式响应类型)

---

### Q: 如何自定义 JSON 序列化？

**A**: 替换 `BodySerializer`。

```ts
import { Plugin, Registry, BodySerializer } from "@aptx/api-core";

const customDateSerializerPlugin: Plugin = {
  setup(registry: Registry) {
    const customBodySerializer: BodySerializer = {
      serialize(req, ctx) {
        if (req.body && typeof req.body === "object") {
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

详见：[extension-points.md - BodySerializer](references/extension-points.md#3-bodyserializer---请求体序列化)
