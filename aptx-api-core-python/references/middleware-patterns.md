# Middleware Patterns

## 目录

- [核心结构](#核心结构)
- [常见模式](#常见模式)
  - [1. 日志 Middleware](#1-日志-middleware)
  - [2. 请求头增强 Middleware](#2-请求头增强-middleware)
  - [3. 错误重试 Middleware](#3-错误重试-middleware)
  - [4. 缓存 Middleware](#4-缓存-middleware简化版)
  - [5. 请求限流 Middleware](#5-请求限流-middleware)
  - [6. 请求去重 Middleware](#6-请求去重-middleware)
  - [7. 业务错误拦截 Middleware](#7-业务错误拦截-middleware)
  - [8. 全局错误处理 Middleware](#8-全局错误处理-middleware)
- [执行顺序](#执行顺序)
- [最佳实践](#最佳实践)

## 核心结构

```python
from aptx_api_core import Middleware

class MyMiddleware:
    async def handle(self, req, ctx, next):
        # 1. 前置处理
        modified_req = req.with_headers({...})

        # 2. 调用下一个 handler
        res = await next(modified_req, ctx)

        # 3. 后置处理
        return res
```

三个参数的含义：
- `req` — 当前请求（`Request` frozen dataclass），通过 `req.with_headers()` 创建修改后的副本
- `ctx` — 请求上下文，包含 `id`、`attempt`、`start_time` 和自由字典 `bag`
- `next` — 下一个处理器，调用后返回 `Response`

## 常见模式

### 1. 日志 Middleware

```python
import time

class LoggingMiddleware:
    async def handle(self, req, ctx, next):
        start = time.monotonic()
        print(f"[{ctx.id}] --> {req.method} {req.url}")

        try:
            res = await next(req, ctx)
            duration_ms = (time.monotonic() - start) * 1000
            print(f"[{ctx.id}] <-- {res.status} {req.url} ({duration_ms:.1f}ms)")
            return res
        except Exception as e:
            duration_ms = (time.monotonic() - start) * 1000
            print(f"[{ctx.id}] !!! {req.url} ({duration_ms:.1f}ms) {type(e).__name__}: {e}")
            raise
```

### 2. 请求头增强 Middleware

```python
import uuid

class CorrelationIdMiddleware:
    async def handle(self, req, ctx, next):
        req = req.with_headers({"X-Correlation-ID": uuid.uuid4().hex})
        return await next(req, ctx)
```

### 3. 错误重试 Middleware

```python
import asyncio
from aptx_api_core import HttpError

class RetryMiddleware:
    def __init__(self, max_retries: int = 3, base_delay: float = 0.5):
        self.max_retries = max_retries
        self.base_delay = base_delay

    async def handle(self, req, ctx, next):
        last_error = None
        for attempt in range(self.max_retries + 1):
            try:
                return await next(req, ctx)
            except (HttpError, TimeoutError, NetworkError) as e:
                last_error = e
                if attempt < self.max_retries:
                    delay = self.base_delay * (2 ** attempt)
                    print(f"重试 {attempt + 1}/{self.max_retries}, 等待 {delay:.1f}s")
                    await asyncio.sleep(delay)
        raise last_error
```

### 4. 缓存 Middleware（简化版）

```python
import hashlib, json

class CacheMiddleware:
    def __init__(self, ttl_seconds: float = 60.0):
        self.ttl = ttl_seconds
        self._cache: dict = {}

    def _key(self, req) -> str:
        raw = f"{req.method}:{req.url}:{json.dumps(req.query or {}, sort_keys=True)}"
        return hashlib.sha256(raw.encode()).hexdigest()

    async def handle(self, req, ctx, next):
        if req.method != "GET":
            return await next(req, ctx)

        key = self._key(req)
        cached = self._cache.get(key)
        if cached and time.monotonic() - cached["time"] < self.ttl:
            ctx.bag["from_cache"] = True
            return cached["response"]

        res = await next(req, ctx)
        self._cache[key] = {"response": res, "time": time.monotonic()}
        return res
```

### 5. 请求限流 Middleware

```python
import asyncio
import time

class RateLimitMiddleware:
    def __init__(self, max_concurrent: int = 5, window_seconds: float = 1.0):
        self.max_concurrent = max_concurrent
        self.window = window_seconds
        self._timestamps: list[float] = []
        self._lock = asyncio.Lock()

    async def handle(self, req, ctx, next):
        async with self._lock:
            now = time.monotonic()
            self._timestamps = [t for t in self._timestamps if now - t < self.window]
            if len(self._timestamps) >= self.max_concurrent:
                raise RuntimeError("Rate limit exceeded")
            self._timestamps.append(now)

        return await next(req, ctx)
```

### 6. 请求去重 Middleware

```python
class DeduplicateMiddleware:
    def __init__(self):
        self._pending: dict[str, asyncio.Future] = {}

    async def handle(self, req, ctx, next):
        if req.method != "GET":
            return await next(req, ctx)

        key = f"{req.method}:{req.url}"

        if key in self._pending:
            return await self._pending[key]

        loop = asyncio.get_event_loop()
        future = loop.create_future()
        self._pending[key] = future

        try:
            res = await next(req, ctx)
            future.set_result(res)
            return res
        except Exception as e:
            future.set_exception(e)
            raise
        finally:
            self._pending.pop(key, None)
```

### 7. 业务错误拦截 Middleware

```python
from aptx_api_core import HttpError

class BusinessErrorMiddleware:
    async def handle(self, req, ctx, next):
        res = await next(req, ctx)

        # 检查业务层错误码
        if isinstance(res.data, dict) and res.data.get("code") != 0:
            error_msg = res.data.get("message", "Unknown business error")
            error_code = res.data.get("code")
            raise RuntimeError(f"业务错误 [{error_code}]: {error_msg}")

        return res
```

### 8. 全局错误处理 Middleware

```python
from aptx_api_core import HttpError, NetworkError, TimeoutError

class GlobalErrorHandlerMiddleware:
    async def handle(self, req, ctx, next):
        try:
            return await next(req, ctx)
        except HttpError as e:
            if e.status == 401:
                print("未授权，需要重新登录")
            elif e.status == 403:
                print("无权限访问")
            elif e.status >= 500:
                print(f"服务端错误: {e.status} {e.url}")
            raise
        except NetworkError:
            print("网络不可用")
            raise
        except TimeoutError:
            print("请求超时")
            raise
```

## 执行顺序

中间件按注册顺序形成洋葱模型：

```
注册顺序: [auth, logging, retry]

请求进入:
  auth.handle → logging.handle → retry.handle → [最终执行 HTTP]

响应返回:
  auth.handle ← logging.handle ← retry.handle ← [最终执行 HTTP]
```

```python
client._client.pipeline.use(auth_middleware)      # 最外层
client._client.pipeline.use(logging_middleware)    # 中间层
client._client.pipeline.use(retry_middleware)      # 最内层
```

## 最佳实践

- **不要在中间件中执行耗时阻塞操作**：所有 I/O 使用 `await`
- **修改请求用 `with_headers()`**：`Request` 是 frozen 的，不能直接赋值
- **用 `ctx.bag` 传数据，不用全局变量**：每次请求有独立的 `Context`
- **异常要 raise 不要吞掉**：除非中间件明确要处理该错误（如重试），否则向上传播
- **认证放最外层**：确保所有请求都经过认证检查
- **日志放中间层**：在认证之后、业务逻辑之前记录
