---
name: aptx-api-core-python
description: "创建、配置或扩展 Python 异步 HTTP 客户端时使用。适用于：使用 aptx-api-core 创建 ApiClient / RequestClient、配置 base_url / timeout / headers、编写 Middleware 拦截请求和响应、添加认证（Bearer / API Key）、统一处理 HTTP 错误和网络异常、在多个中间件之间通过 Context 共享数据、为代码生成的 Python API 客户端配置运行时。当用户提到 Python 异步 HTTP 客户端封装、请求拦截器/管道、错误重试与超时、httpx 封装层定制、生成的 Python SDK 运行时配置、请求认证中间件时，即使未提及 aptx-api-core 也应触发。"
---

# aptx-api-core-python

Python 异步 HTTP 运行时，为 aptx 代码生成的 API 客户端提供请求执行能力。

在需要接入或调整请求内核时，按以下顺序执行：

1. 创建 `RequestClient`，确定全局配置：`base_url`、`headers`、`timeout`。详见 [实例化配置](#实例化配置)
2. 通过 `pipeline.use(middleware)` 注册中间件处理横切关注点（认证、日志、错误处理等）。详见 [中间件](#中间件)
3. 使用 `AuthMiddleware` 或自定义中间件添加认证信息。详见 [授权与认证](#授权与认证)
4. 在调用方使用 `try/except` 按错误类型分流：`HttpError`、`NetworkError`、`TimeoutError`、`DecodeError`。详见 [错误处理](#错误处理)
5. 通过全局单例 `set_api_client()` / `get_api_client()` 让生成代码访问共享客户端。详见 [全局单例](#全局单例)

最小接入模板：

```python
from aptx_api_core import (
    create_api_client, set_api_client, get_api_client,
    RequestSpec, AuthMiddleware,
)

# 创建并注册全局客户端
client = create_api_client(base_url="https://api.example.com", timeout=10.0)
set_api_client(client)

# 发送请求
spec = RequestSpec(method="GET", path="/user/123")
user = await get_api_client().execute_async(spec, response_type=UserDto)
```

## 快速参考

| 操作 | API | 示例 |
|------|-----|------|
| 创建客户端 | `create_api_client()` | 见上方模板 |
| 发送请求 | `client.execute_async(spec, response_type)` | `await client.execute_async(spec, UserDto)` |
| 添加 middleware | `client.pipeline.use(mw)` | `client.pipeline.use(auth_mw)` |
| 认证中间件 | `AuthMiddleware(get_token=...)` | 见[授权与认证](#授权与认证) |
| 全局单例 | `set_api_client()` / `get_api_client()` | 见[全局单例](#全局单例) |
| 错误拦截 | `try/except UniReqError` | 见[错误处理](#错误处理) |

### 核心 API 映射

| 需求 | 方法 | 文档位置 |
|------|------|----------|
| 请求/响应修改 | Middleware | [中间件](#中间件) |
| 认证（Bearer / API Key） | AuthMiddleware | [授权与认证](#授权与认证) |
| 统一错误处理 | try/except + 错误层级 | [错误处理](#错误处理) |
| 中间件间共享状态 | Context.bag | [Context 与状态共享](#context-与状态共享) |
| 自定义中间件模式 | Middleware patterns | [references/middleware-patterns.md](references/middleware-patterns.md) |
| 测试 | mock httpx | [references/testing-guide.md](references/testing-guide.md) |

## 实例化配置

```python
from aptx_api_core import RequestClient, ApiClient

# 方式一：直接创建 RequestClient（完全控制）
inner = RequestClient(
    base_url="https://api.example.com",
    headers={"X-App": "web"},
    timeout=10.0,
    middlewares=[logging_middleware],  # 初始中间件列表
)
client = ApiClient(inner)

# 方式二：工厂函数（推荐）
from aptx_api_core import create_api_client

client = create_api_client(
    base_url="https://api.example.com",
    timeout=10.0,
    middlewares=[logging_middleware],
)
```

### RequestClient 构造参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `base_url` | `str` | 否 | `""` | API 基础 URL |
| `headers` | `dict[str, str]` | 否 | `None` | 默认请求头 |
| `timeout` | `float` | 否 | `None` | 超时秒数 |
| `middlewares` | `list` | 否 | `None` | 初始中间件列表 |

### execute_async 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `spec` | `RequestSpec` | 是 | - | 请求描述 |
| `response_type` | `type` | 否 | `None` | 响应类型（自动实例化） |
| `options` | `PerCallOptions` | 否 | `None` | 单次调用覆盖选项 |

## 中间件

中间件是处理横切关注点的核心机制。每个中间件接收 `req`（Request）、`ctx`（Context）和 `next`（下一个处理器），可在请求前后插入逻辑，或完全短路请求链。

### Middleware 协议

```python
from typing import Protocol, Awaitable, Callable
from aptx_api_core import Middleware

class Middleware(Protocol):
    async def handle(
        self,
        req: Request,
        ctx: Context,
        next: Callable[[Request, Context], Awaitable[Response]],
    ) -> Response: ...
```

### 注册中间件

```python
from aptx_api_core import create_api_client, set_api_client

client = create_api_client(base_url="https://api.example.com")

# 在 pipeline 上注册
client._client.pipeline.use(logging_middleware)
client._client.pipeline.use(auth_middleware)
client._client.pipeline.use(error_handling_middleware)

set_api_client(client)
```

> `create_api_client()` 返回 `ApiClient`，它内部包装了 `RequestClient`。通过 `client._client` 访问内部的 `RequestClient` 来操作 pipeline。也可以在创建时通过 `middlewares` 参数直接传入。

> 中间件按注册顺序执行：先注册的先进入，后返回（洋葱模型）。最外层的中间件最先执行前置逻辑、最后执行后置逻辑——因此认证应放最外层，日志放中间层。

### 常见中间件模式

日志中间件示例：

```python
from aptx_api_core import Middleware

class LoggingMiddleware:
    async def handle(self, req, ctx, next):
        print(f"[{req.method}] {req.url}")
        res = await next(req, ctx)
        print(f"[{res.status}] {req.url}")
        return res
```

更多模式（缓存、限流、去重、重试、签名等）详见 [references/middleware-patterns.md](references/middleware-patterns.md)。

## 授权与认证

`AuthMiddleware` 是内置的认证中间件，支持 Bearer Token 和自定义 Header 认证。

### Bearer Token 认证

```python
from aptx_api_core import create_api_client, AuthMiddleware, set_api_client

# 定义 token 获取函数
async def get_token() -> str | None:
    # 从环境变量、配置文件或 token store 读取
    import os
    return os.environ.get("API_TOKEN")

auth = AuthMiddleware(get_token=get_token)

client = create_api_client(base_url="https://api.example.com")
client._client.pipeline.use(auth)
set_api_client(client)
```

### API Key 认证（自定义 Header）

```python
auth = AuthMiddleware(
    get_token=get_api_key,
    header_name="X-API-Key",
    token_prefix="",  # 不添加 "Bearer " 前缀
)
```

### AuthMiddleware 配置选项

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `get_token` | `async () -> str \| None` | 是 | - | 获取 token 的异步函数 |
| `header_name` | `str` | 否 | `"Authorization"` | 目标 Header 名称 |
| `token_prefix` | `str` | 否 | `"Bearer "` | Token 前缀 |

### 动态 Token 刷新

结合 token store 实现 token 过期时自动刷新：

```python
import time

_token_cache: dict = {"token": None, "expires_at": 0}

async def get_token_with_refresh() -> str | None:
    if _token_cache["token"] and time.time() < _token_cache["expires_at"]:
        return _token_cache["token"]

    # 刷新 token（调用登录接口等）
    new_token, expires_in = await refresh_token()
    _token_cache["token"] = new_token
    _token_cache["expires_at"] = time.time() + expires_in
    return new_token

auth = AuthMiddleware(get_token=get_token_with_refresh)
```

### 自定义认证中间件

当需要更复杂的认证逻辑（如 HMAC 签名、OAuth）时，直接编写中间件：

```python
import hmac, hashlib, time

class HmacSignMiddleware:
    def __init__(self, secret_key: str):
        self.secret_key = secret_key

    async def handle(self, req, ctx, next):
        timestamp = str(int(time.time()))
        message = f"{req.method}{req.url}{timestamp}"
        signature = hmac.new(
            self.secret_key.encode(), message.encode(), hashlib.sha256
        ).hexdigest()
        req = req.with_headers({
            "X-Timestamp": timestamp,
            "X-Signature": signature,
        })
        return await next(req, ctx)
```

## 错误处理

### 错误层级

```
UniReqError (基类)
├── NetworkError      — 网络层错误（DNS 失败、连接拒绝）
├── TimeoutError      — 请求超时
├── HttpError         — HTTP 4xx / 5xx 响应
└── DecodeError       — 响应体解码失败
```

### 错误属性

| 错误类型 | 属性 | 说明 |
|---------|------|------|
| `HttpError` | `.status` | HTTP 状态码 |
| | `.url` | 请求 URL |
| | `.body_preview` | 响应体前 500 字符 |
| `DecodeError` | `.response_type` | 预期响应类型 |
| | `.status` | HTTP 状态码 |
| | `.url` | 请求 URL |
| `NetworkError` | — | 错误信息在 `str(e)` 中 |
| `TimeoutError` | — | 错误信息在 `str(e)` 中 |

### 基本错误捕获

```python
from aptx_api_core import (
    get_api_client, HttpError, NetworkError, TimeoutError, DecodeError, UniReqError,
)

try:
    user = await get_api_client().execute_async(spec, response_type=UserDto)
except HttpError as e:
    if e.status == 404:
        print(f"用户不存在: {e.url}")
    elif e.status == 401:
        print(f"未授权: {e.url}")
    else:
        print(f"HTTP 错误 {e.status}: {e.url}, body={e.body_preview}")
except NetworkError:
    print("网络不可用，请检查连接")
except TimeoutError:
    print("请求超时，请稍后重试")
except DecodeError as e:
    print(f"响应解码失败: {e.response_type}, status={e.status}")
except UniReqError as e:
    print(f"未知请求错误: {e}")
```

### 中间件内统一错误处理

```python
from aptx_api_core import HttpError

class ErrorHandlerMiddleware:
    async def handle(self, req, ctx, next):
        try:
            return await next(req, ctx)
        except HttpError as e:
            if e.status == 401:
                # 跳转登录页
                raise
            if e.status >= 500:
                # 服务端错误，记录日志
                print(f"Server error: {e.status} {e.url}")
                raise
            raise
        except Exception as e:
            print(f"Unexpected error: {e}")
            raise
```

### 错误映射机制

底层 `httpx` 异常会被自动映射为 `aptx_api_core` 错误：

| httpx 异常 | 映射为 |
|-----------|--------|
| `httpx.TimeoutException` | `TimeoutError` |
| `httpx.ConnectError` | `NetworkError` |
| `httpx.HTTPStatusError` | `HttpError` |
| 其他异常 | `UniReqError` |

已有的 `UniReqError` 子类会原样透传，不会被二次包装。

## Context 与状态共享

每次请求自动创建一个 `Context` 对象，通过 `ctx.bag` 在中间件间传递数据：

```python
@dataclass
class Context:
    id: str          # 随机 16 位 hex
    attempt: int     # 重试计数
    start_time: float  # 请求开始时间
    bag: dict        # 自由数据字典
```

### 使用示例

```python
class TimingMiddleware:
    async def handle(self, req, ctx, next):
        ctx.bag["step1_time"] = time.monotonic()
        res = await next(req, ctx)
        duration = time.monotonic() - ctx.bag["step1_time"]
        ctx.bag["total_duration_ms"] = duration * 1000
        return res

class AuditMiddleware:
    async def handle(self, req, ctx, next):
        res = await next(req, ctx)
        # 读取前一个中间件写入的数据
        duration = ctx.bag.get("total_duration_ms", 0)
        print(f"[{ctx.id}] {req.method} {req.url} - {duration:.1f}ms")
        return res
```

## 全局单例

生成代码通常通过 `get_api_client()` 访问全局客户端。应用启动时注册：

```python
from aptx_api_core import create_api_client, set_api_client, AuthMiddleware

# 应用初始化
async def init_api():
    async def get_token() -> str | None:
        return get_token_from_store()

    auth = AuthMiddleware(get_token=get_token)
    client = create_api_client(
        base_url="https://api.example.com",
        timeout=10.0,
        middlewares=[auth],
    )
    set_api_client(client)
```

> `get_api_client()` 在未初始化时抛出 `RuntimeError`。

## RequestSpec

`RequestSpec` 是生成代码构造的请求描述对象：

```python
from aptx_api_core import RequestSpec

spec = RequestSpec(
    method="POST",
    path="/api/users/{id}",
    query={"include": "profile"},
    headers={"X-Custom": "value"},
    body={"name": "Alice"},
    input=my_input_object,  # 用于路径参数替换
    meta={"tag": "user-create"},
)
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `method` | `HttpMethod` | HTTP 方法 |
| `path` | `str` | 请求路径（支持 `{param}` 模板） |
| `query` | `dict \| None` | 查询参数 |
| `headers` | `dict \| None` | 请求头 |
| `body` | `Any` | 请求体 |
| `input` | `Any` | 输入对象（用于路径参数替换） |
| `meta` | `dict \| None` | 扩展元数据 |

路径参数替换规则：`DefaultUrlResolver` 自动将 `{paramName}` 替换为 `input.paramName` 的值。

## PerCallOptions

单次调用的覆盖选项，不修改全局配置：

```python
from aptx_api_core import PerCallOptions

opts = PerCallOptions(
    headers={"X-Request-ID": "abc123"},
    query={"version": "2"},
    timeout=5.0,
    meta={"priority": "high"},
)
await client.execute_async(spec, options=opts)
```

## 关键约束

- 不在 core 层处理业务逻辑（重试、缓存等通过中间件实现）——保持核心轻量，所有横切关注点由使用者通过中间件按需组合
- `Request` 是 frozen dataclass，修改用 `req.with_headers({...})` 返回新实例——避免中间件并发修改同一请求对象时产生竞态条件
- 中间件按注册顺序执行（洋葱模型：先进后出）——认证放最外层，日志放中间层，确保日志能记录认证后的状态
- 全局单例 `get_api_client()` 必须在 `set_api_client()` 之后调用，否则抛出 `RuntimeError`
- 需要 Python >=3.10，依赖 `httpx` >=0.27

## 何时不使用

- **同步代码**：此库完全基于 `async/await`，不适用于同步场景
- **非 HTTP 通信**：gRPC、WebSocket 等协议不在支持范围
- **简单脚本**：仅需一次性 HTTP 调用时，直接使用 `httpx` 更简洁

## 参考文档

| 主题 | 文件 | 说明 |
|------|------|------|
| 中间件模式 | [references/middleware-patterns.md](references/middleware-patterns.md) | 日志、缓存、限流、签名等常见模式 |
| 测试指南 | [references/testing-guide.md](references/testing-guide.md) | 使用 respx mock HTTP 请求的方法 |
