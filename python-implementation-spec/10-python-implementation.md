# Python Implementation Guide

## Technology Stack

### Core Framework: FastAPI

**Why FastAPI**:
- Native async/await support (required for streaming)
- Automatic request/response validation with Pydantic
- Built-in OpenAPI documentation
- Excellent performance (comparable to Node.js)
- Great developer experience
- Easy testing with TestClient

### Required Libraries

```toml
[project]
name = "claude-code-router"
version = "1.0.0"
dependencies = [
    "fastapi>=0.100.0",
    "uvicorn[standard]>=0.23.0",  # ASGI server
    "httpx>=0.24.0",              # Async HTTP client
    "pydantic>=2.0.0",            # Data validation
    "click>=8.0.0",               # CLI framework
    "tiktoken>=0.5.0",            # Token counting
    "pyjson5>=1.6.0",             # JSON5 parsing
    "python-multipart>=0.0.6",    # File uploads
]

[project.optional-dependencies]
mlx = [
    "mlx>=0.4.0",                 # Apple MLX framework
    "mlx-lm>=0.4.0",              # MLX language models
]

[project.scripts]
ccr = "claude_code_router.cli:cli"
```

## Project Structure

```
claude-code-router/
├── README.md
├── pyproject.toml
├── setup.py
├── src/
│   └── claude_code_router/
│       ├── __init__.py
│       ├── cli.py               # CLI entry point
│       ├── server.py            # FastAPI server
│       ├── config/
│       │   ├── __init__.py
│       │   ├── manager.py       # Config loading/saving
│       │   └── models.py        # Pydantic models
│       ├── routing/
│       │   ├── __init__.py
│       │   ├── router.py        # Routing logic
│       │   ├── token_counter.py # Token counting
│       │   └── session_cache.py # LRU cache
│       ├── providers/
│       │   ├── __init__.py
│       │   ├── client.py        # HTTP client
│       │   ├── base.py          # Base classes
│       │   └── transformers/
│       │       ├── __init__.py
│       │       ├── anthropic.py
│       │       ├── deepseek.py
│       │       ├── gemini.py
│       │       └── ...
│       ├── agents/
│       │   ├── __init__.py
│       │   ├── base.py          # Agent interface
│       │   ├── manager.py       # Agent manager
│       │   └── image_agent.py   # Image agent
│       ├── streaming/
│       │   ├── __init__.py
│       │   ├── sse_parser.py
│       │   ├── sse_serializer.py
│       │   └── stream_utils.py
│       ├── middleware/
│       │   ├── __init__.py
│       │   ├── auth.py
│       │   ├── logging.py
│       │   └── router.py
│       ├── api/
│       │   ├── __init__.py
│       │   ├── messages.py      # /v1/messages endpoint
│       │   ├── config.py        # /api/config endpoints
│       │   └── logs.py          # /api/logs endpoints
│       ├── mlx/                 # MLX integration (optional)
│       │   ├── __init__.py
│       │   ├── loader.py
│       │   └── server.py
│       └── utils/
│           ├── __init__.py
│           ├── process.py       # PID management
│           ├── logging.py
│           └── statusline.py
└── tests/
    ├── __init__.py
    ├── test_routing.py
    ├── test_transformers.py
    ├── test_agents.py
    └── ...
```

## Implementation Example

### 1. Main Server (`server.py`)

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from fastapi.staticfiles import StaticFiles
import uvicorn

from .config import Config, ConfigManager
from .middleware import AuthMiddleware, LoggingMiddleware, RouterMiddleware
from .api import messages, config, logs
from .agents import AgentsManager
from .routing import SessionCache

async def create_app() -> FastAPI:
    """Create and configure FastAPI application."""
    # Load config
    config_manager = ConfigManager()
    config = await config_manager.load()

    # Create app
    app = FastAPI(
        title="Claude Code Router",
        description="Route Claude Code requests to different LLM providers",
        version="1.0.0"
    )

    # State
    app.state.config = config
    app.state.config_manager = config_manager
    app.state.session_cache = SessionCache(capacity=100)
    app.state.agents_manager = AgentsManager()

    # Middleware (order matters!)
    app.add_middleware(LoggingMiddleware)
    app.add_middleware(AuthMiddleware, config=config)
    app.add_middleware(RouterMiddleware, config=config)

    # Routes
    app.include_router(messages.router, prefix="/v1", tags=["messages"])
    app.include_router(config.router, prefix="/api", tags=["config"])
    app.include_router(logs.router, prefix="/api", tags=["logs"])

    # Static files (UI)
    # app.mount("/ui", StaticFiles(directory="dist", html=True), name="ui")

    # Health check
    @app.get("/")
    @app.get("/health")
    async def health():
        return {"status": "ok"}

    return app

async def start_server(config: Config):
    """Start the server."""
    app = await create_app()

    uvicorn.run(
        app,
        host=config.HOST,
        port=config.PORT,
        log_level=config.LOG_LEVEL.lower() if config.LOG else "error"
    )
```

### 2. Messages Endpoint (`api/messages.py`)

```python
from fastapi import APIRouter, Request, HTTPException
from fastapi.responses import StreamingResponse

from ..providers import load_transformers, call_provider_api
from ..streaming import SSEParser, SSESerializer

router = APIRouter()

@router.post("/messages")
async def handle_messages(request: Request):
    """Main message handling endpoint."""
    body = await request.json()

    # Model was selected by RouterMiddleware
    model = body.get("model")
    if not model or "," not in model:
        raise HTTPException(status_code=400, detail="Invalid model format")

    provider_name, model_name = model.split(",", 1)

    # Get provider config
    config = request.app.state.config
    provider = next(
        (p for p in config.Providers if p.name == provider_name),
        None
    )
    if not provider:
        raise HTTPException(status_code=400, detail=f"Provider not found: {provider_name}")

    # Load transformers
    transformers = load_transformers(provider, model_name)

    # Transform request
    transformed_request = await transformers.transform_request(body)

    # Call provider
    stream = body.get("stream", False)
    response = await call_provider_api(
        provider=provider,
        model=model_name,
        request_body=transformed_request,
        stream=stream,
        timeout=config.API_TIMEOUT_MS
    )

    # Return response
    if stream:
        async def generate():
            # Parse SSE
            parser = SSEParser()
            events = parser.parse_stream(response.aiter_bytes())

            # Transform events
            async for event in events:
                transformed = await transformers.transform_response(event)
                sse_text = SSESerializer.serialize(transformed)
                yield sse_text

        return StreamingResponse(generate(), media_type="text/event-stream")
    else:
        response_data = await response.json()
        transformed_response = await transformers.transform_response(response_data)
        return transformed_response

@router.post("/messages/count_tokens")
async def count_tokens(request: Request):
    """Count tokens in a request."""
    from ..routing import calculate_token_count

    body = await request.json()
    count = calculate_token_count(
        messages=body.get("messages", []),
        system=body.get("system", []),
        tools=body.get("tools", [])
    )
    return {"input_tokens": count}
```

### 3. Router Middleware (`middleware/router.py`)

```python
from starlette.middleware.base import BaseHTTPMiddleware
from ..routing import get_use_model, extract_session_id, calculate_token_count

class RouterMiddleware(BaseHTTPMiddleware):
    """Routing middleware."""

    def __init__(self, app, config):
        super().__init__(app)
        self.config = config

    async def dispatch(self, request: Request, call_next):
        # Only route /v1/messages requests
        if not request.url.path.startswith("/v1/messages"):
            return await call_next(request)

        if request.url.path == "/v1/messages/count_tokens":
            return await call_next(request)

        # Get request body
        body = await request.json()

        # Extract session ID
        session_id = extract_session_id(body)
        request.state.session_id = session_id

        # Get last usage
        session_cache = request.app.state.session_cache
        last_usage = session_cache.get(session_id) if session_id else None

        # Count tokens
        token_count = calculate_token_count(
            messages=body.get("messages", []),
            system=body.get("system", []),
            tools=body.get("tools", [])
        )
        request.state.token_count = token_count

        # Select model
        model = await get_use_model(request, token_count, self.config, last_usage)
        body["model"] = model

        # Store modified body
        request.state.body = body

        return await call_next(request)
```

### 4. CLI (`cli.py`)

```python
import click
import asyncio
from .server import start_server
from .config import ConfigManager

@click.group()
def cli():
    """Claude Code Router CLI"""
    pass

@cli.command()
def start():
    """Start the router service"""
    from .lifecycle import cmd_start
    asyncio.run(cmd_start())

@cli.command()
def stop():
    """Stop the router service"""
    from .lifecycle import cmd_stop
    cmd_stop()

@cli.command()
@click.argument('prompt', nargs=-1)
def code(prompt):
    """Execute Claude Code through the router"""
    from .lifecycle import cmd_code
    asyncio.run(cmd_code(list(prompt)))

# ... more commands

if __name__ == "__main__":
    cli()
```

## Testing

```python
# tests/test_messages.py
import pytest
from httpx import AsyncClient
from claude_code_router.server import create_app

@pytest.fixture
async def app():
    return await create_app()

@pytest.fixture
async def client(app):
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.mark.asyncio
async def test_count_tokens(client):
    response = await client.post("/v1/messages/count_tokens", json={
        "messages": [{"role": "user", "content": "Hello"}],
        "system": [],
        "tools": []
    })
    assert response.status_code == 200
    assert "input_tokens" in response.json()

@pytest.mark.asyncio
async def test_streaming_messages(client):
    async with client.stream(
        "POST",
        "/v1/messages",
        json={
            "model": "test,model",
            "messages": [{"role": "user", "content": "Hello"}],
            "stream": True
        }
    ) as response:
        events = []
        async for line in response.aiter_lines():
            if line.startswith("data: "):
                events.append(line)
        assert len(events) > 0
```

## Performance Considerations

### Async Best Practices

- Use `async`/`await` throughout
- Don't block the event loop
- Use `asyncio.gather()` for parallel operations
- Connection pooling for HTTP clients

### Caching

- LRU cache for session usage
- LRU cache for images (agent system)
- Consider caching transformer instances

### Logging

- Use structured logging (JSON)
- Log levels: DEBUG for development, INFO for production
- Rotating logs to prevent disk fill-up

## Deployment

### Docker

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN pip install -e .

EXPOSE 3456

CMD ["ccr", "start"]
```

### Systemd Service

```ini
[Unit]
Description=Claude Code Router
After=network.target

[Service]
Type=simple
User=user
WorkingDirectory=/home/user
ExecStart=/usr/local/bin/ccr start
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
