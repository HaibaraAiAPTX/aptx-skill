# Testing Guide

## 目录

- [核心思路](#核心思路)
- [使用 respx Mock HTTP](#使用-respx-mock-http)
- [测试中间件](#测试中间件)
- [测试错误处理](#测试错误处理)
- [测试全局单例](#测试全局单例)
- [完整示例](#完整示例)

## 核心思路

`aptx-api-core` 基于 `httpx`，推荐使用 `respx` 库 mock HTTP 请求，无需启动真实服务器。

依赖安装：

```bash
pip install pytest pytest-asyncio respx
```

## 使用 respx Mock HTTP

```python
import pytest
import respx
from aptx_api_core import create_api_client, RequestSpec

# 假设 UserDto 是代码生成的 Pydantic 模型，或自行定义的响应类型
@pydantic.dataclasses.dataclass
class UserDto:
    id: str
    name: str

@respx.mock
@pytest.mark.asyncio
async def test_get_user():
    # 模拟 GET 请求
    respx.get("https://api.example.com/user/123").mock(
        return_value=httpx.Response(200, json={"id": "123", "name": "Alice"})
    )

    client = create_api_client(base_url="https://api.example.com")
    spec = RequestSpec(method="GET", path="/user/123")
    result = await client.execute_async(spec, response_type=UserDto)

    assert result.name == "Alice"
```

## 测试中间件

```python
import pytest
from aptx_api_core import create_api_client, RequestSpec

class HeaderSpyMiddleware:
    """记录请求头的测试中间件"""
    def __init__(self):
        self.captured_headers = {}

    async def handle(self, req, ctx, next):
        self.captured_headers = dict(req.headers)
        return await next(req, ctx)

@pytest.mark.asyncio
async def test_auth_middleware_adds_header():
    spy = HeaderSpyMiddleware()

    async def get_token():
        return "test-token"

    from aptx_api_core import AuthMiddleware
    auth = AuthMiddleware(get_token=get_token)

    client = create_api_client(base_url="https://api.example.com")
    client._client.pipeline.use(spy)
    client._client.pipeline.use(auth)

    # 用 respx mock 实际请求
    with respx.mock:
        respx.get("https://api.example.com/test").mock(
            return_value=httpx.Response(200, json={})
        )
        spec = RequestSpec(method="GET", path="/test")
        await client.execute_async(spec)

    assert "Authorization" in spy.captured_headers
    assert spy.captured_headers["Authorization"] == "Bearer test-token"
```

## 测试错误处理

```python
import pytest
import httpx
import respx
from aptx_api_core import (
    create_api_client, RequestSpec,
    HttpError, NetworkError, TimeoutError, DecodeError,
)

@respx.mock
@pytest.mark.asyncio
async def test_http_404():
    respx.get("https://api.example.com/user/999").mock(
        return_value=httpx.Response(404, text="Not Found")
    )

    client = create_api_client(base_url="https://api.example.com")
    spec = RequestSpec(method="GET", path="/user/999")

    with pytest.raises(HttpError) as exc_info:
        await client.execute_async(spec)

    assert exc_info.value.status == 404


@respx.mock
@pytest.mark.asyncio
async def test_timeout():
    respx.get("https://api.example.com/slow").mock(
        side_effect=httpx.ReadTimeout("timeout")
    )

    client = create_api_client(base_url="https://api.example.com")
    spec = RequestSpec(method="GET", path="/slow")

    with pytest.raises(TimeoutError):
        await client.execute_async(spec)
```

## 测试全局单例

```python
import pytest
from aptx_api_core import set_api_client, get_api_client, create_api_client

def test_global_client_not_set():
    set_api_client(None)  # 重置
    with pytest.raises(RuntimeError, match="ApiClient not initialized"):
        get_api_client()

def test_set_and_get():
    client = create_api_client(base_url="https://api.example.com")
    set_api_client(client)
    assert get_api_client() is client
```

## 完整示例

```python
import pytest
import httpx
import respx
from aptx_api_core import (
    create_api_client, set_api_client, get_api_client,
    RequestSpec, AuthMiddleware, HttpError,
)

@respx.mock
@pytest.mark.asyncio
async def test_full_workflow():
    # 1. 模拟 API 响应
    respx.get("https://api.example.com/user/123").mock(
        return_value=httpx.Response(200, json={"id": "123", "name": "Alice"})
    )

    # 2. 配置客户端
    async def get_token():
        return "test-token"

    auth = AuthMiddleware(get_token=get_token)
    client = create_api_client(
        base_url="https://api.example.com",
        middlewares=[auth],
    )
    set_api_client(client)

    # 3. 执行请求
    spec = RequestSpec(method="GET", path="/user/123")
    result = await get_api_client().execute_async(spec)

    # 4. 验证结果
    assert result["name"] == "Alice"
```
