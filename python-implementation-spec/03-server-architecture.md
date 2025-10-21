# Server Architecture Specification

## Overview

The server is an HTTP service based on Fastify (in TypeScript) that provides an Anthropic-compatible API endpoint along with management endpoints. It uses hooks/middleware for request processing and supports streaming responses.

## Server Framework

**TypeScript implementation**: Fastify (via `@musistudio/llms` package)

**Python recommendation**: FastAPI or Starlette
- FastAPI: Full-featured, automatic OpenAPI docs, dependency injection
- Starlette: Lightweight, async, good streaming support

## Server Lifecycle

### Initialization Flow

```python
async def create_server(config: Config) -> Server:
    """
    Creates and configures the HTTP server.

    Args:
        config: Application configuration

    Returns:
        Configured server instance
    """
    # 1. Initialize FastAPI/Starlette app
    from fastapi import FastAPI
    from fastapi.responses import StreamingResponse

    app = FastAPI(
        title="Claude Code Router",
        version="1.0.0"
    )

    # 2. Configure logging
    if config.LOG:
        setup_logging(config)

    # 3. Initialize state
    app.state.config = config
    app.state.session_cache = LRUCache(100)
    app.state.image_cache = ImageCache(100)
    app.state.event_emitter = EventEmitter()

    # 4. Register middleware
    register_middleware(app, config)

    # 5. Register routes
    register_routes(app)

    # 6. Setup error handlers
    register_error_handlers(app)

    return app


async def start_server(config: Config):
    """
    Start the HTTP server.
    """
    app = await create_server(config)

    # Get host/port from config
    host = config.HOST or "127.0.0.1"
    port = config.PORT or 3456

    # Force host to 127.0.0.1 if no API key
    if config.HOST and not config.APIKEY:
        host = "127.0.0.1"
        print("⚠️ API key is not set. HOST is forced to 127.0.0.1.")

    # Run server
    import uvicorn
    uvicorn.run(
        app,
        host=host,
        port=port,
        log_level=config.LOG_LEVEL.lower() if config.LOG else "error"
    )
```

### Shutdown Sequence

```python
@app.on_event("shutdown")
async def shutdown_event():
    """
    Cleanup on server shutdown.
    """
    # Close any open connections
    # Flush logs
    # Clean up temporary files
    cleanup_pid_file()
```

## API Endpoints

### 1. Main Message API

**Endpoint**: `POST /v1/messages`

**Purpose**: Process Claude Code message requests

**Request Body** (Anthropic Messages API format):
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "Hello, Claude"
    }
  ],
  "system": [
    {
      "type": "text",
      "text": "You are a helpful assistant..."
    }
  ],
  "stream": true,
  "tools": [...],
  "metadata": {
    "user_id": "user123_session_abc456"
  },
  "thinking": null
}
```

**Response** (streaming SSE if `stream: true`):
```
event: message_start
data: {"type":"message_start","message":{"id":"msg_123",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,...}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":10}}

event: message_stop
data: {"type":"message_stop"}
```

**Implementation**:
```python
@app.post("/v1/messages")
async def handle_messages(request: Request):
    """
    Main message handling endpoint.

    Flow:
    1. Middleware processes request (auth, agent, routing)
    2. Load transformers for selected model
    3. Transform request to provider format
    4. Call provider API
    5. Transform response back to Anthropic format
    6. Stream response
    7. Track usage in session cache
    """
    body = await request.json()

    # Model and provider are set by router middleware
    model = body.get("model")  # Format: "provider,model"
    provider_name, model_name = model.split(",", 1)

    # Get provider config
    provider = next(
        p for p in request.app.state.config.Providers
        if p.name == provider_name
    )

    # Load and apply transformers
    transformers = load_transformers(provider, model_name)
    transformed_request = apply_request_transformers(body, transformers)

    # Call provider API
    response = await call_provider_api(
        provider=provider,
        model=model_name,
        request_body=transformed_request,
        stream=body.get("stream", False)
    )

    # Transform and stream response
    if body.get("stream"):
        return StreamingResponse(
            transform_and_stream_response(response, transformers),
            media_type="text/event-stream"
        )
    else:
        response_data = await response.json()
        return apply_response_transformers(response_data, transformers)
```

### 2. Token Counting API

**Endpoint**: `POST /v1/messages/count_tokens`

**Purpose**: Estimate token count without making a full API call

**Request/Response**:
```python
@app.post("/v1/messages/count_tokens")
async def count_tokens(request: Request):
    """
    Count tokens in a request.
    """
    body = await request.json()

    token_count = calculate_token_count(
        messages=body.get("messages", []),
        system=body.get("system", []),
        tools=body.get("tools", [])
    )

    return {"input_tokens": token_count}
```

### 3. Configuration API

**Endpoint**: `GET /api/config`

**Purpose**: Retrieve current configuration (for UI)

```python
@app.get("/api/config")
async def get_config(request: Request):
    """
    Returns current configuration.
    """
    return request.app.state.config.dict()
```

**Endpoint**: `POST /api/config`

**Purpose**: Update configuration

```python
@app.post("/api/config")
async def save_config(request: Request):
    """
    Save new configuration.
    """
    new_config = await request.json()

    # Backup existing config
    backup_path = await backup_config_file()
    if backup_path:
        print(f"Backed up config to {backup_path}")

    # Write new config
    await write_config_file(new_config)

    # Reload config
    request.app.state.config = await read_config_file()

    return {"success": True, "message": "Config saved successfully"}
```

### 4. Transformers API

**Endpoint**: `GET /api/transformers`

**Purpose**: List available transformers

```python
@app.get("/api/transformers")
async def get_transformers(request: Request):
    """
    Returns list of available transformers.
    """
    # This would query the transformer service
    transformers = get_all_transformers()

    return {
        "transformers": [
            {
                "name": t.name,
                "endpoint": t.endpoint or None
            }
            for t in transformers
        ]
    }
```

### 5. Restart API

**Endpoint**: `POST /api/restart`

**Purpose**: Restart the service

```python
@app.post("/api/restart")
async def restart_service(request: Request):
    """
    Restart the service.
    """
    import os
    import sys
    from subprocess import Popen

    # Send response first
    response = {"success": True, "message": "Service restart initiated"}

    # Schedule restart after response is sent
    async def restart_after_response():
        await asyncio.sleep(1)
        # Restart process
        Popen([sys.executable] + sys.argv)
        os._exit(0)

    asyncio.create_task(restart_after_response())

    return response
```

### 6. Logs API

**Endpoint**: `GET /api/logs/files`

**Purpose**: List available log files

```python
@app.get("/api/logs/files")
async def get_log_files(request: Request):
    """
    Returns list of log files.
    """
    log_dir = Path.home() / ".claude-code-router" / "logs"
    log_files = []

    if log_dir.exists():
        for log_file in log_dir.glob("*.log"):
            stat = log_file.stat()
            log_files.append({
                "name": log_file.name,
                "path": str(log_file),
                "size": stat.st_size,
                "lastModified": stat.st_mtime
            })

    # Sort by modification time (newest first)
    log_files.sort(key=lambda x: x["lastModified"], reverse=True)

    return log_files
```

**Endpoint**: `GET /api/logs?file=/path/to/log`

**Purpose**: Read log file contents

```python
@app.get("/api/logs")
async def get_logs(request: Request, file: str = None):
    """
    Returns log file contents.
    """
    if file:
        log_file = Path(file)
    else:
        log_file = Path.home() / ".claude-code-router" / "logs" / "app.log"

    if not log_file.exists():
        return []

    # Read and return log lines
    lines = log_file.read_text().split("\n")
    return [line for line in lines if line.strip()]
```

**Endpoint**: `DELETE /api/logs?file=/path/to/log`

**Purpose**: Clear log file

```python
@app.delete("/api/logs")
async def clear_logs(request: Request, file: str = None):
    """
    Clear a log file.
    """
    if file:
        log_file = Path(file)
    else:
        log_file = Path.home() / ".claude-code-router" / "logs" / "app.log"

    if log_file.exists():
        log_file.write_text("")

    return {"success": True, "message": "Logs cleared successfully"}
```

### 7. Update Check API

**Endpoint**: `GET /api/update/check`

**Purpose**: Check for updates

```python
@app.get("/api/update/check")
async def check_for_updates(request: Request):
    """
    Check if a newer version is available.
    """
    import httpx

    current_version = "1.0.62"  # From package.json

    # Check PyPI or GitHub for latest version
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.github.com/repos/user/claude-code-router-py/releases/latest"
        )
        data = response.json()
        latest_version = data["tag_name"].lstrip("v")

    has_update = latest_version > current_version

    return {
        "hasUpdate": has_update,
        "latestVersion": latest_version if has_update else None,
        "changelog": data.get("body") if has_update else None
    }
```

### 8. Static File Serving (UI)

**Endpoint**: `GET /ui/*`

**Purpose**: Serve web UI files

```python
from fastapi.staticfiles import StaticFiles

app.mount(
    "/ui",
    StaticFiles(directory="dist", html=True),
    name="ui"
)

# Redirect /ui to /ui/
@app.get("/ui")
async def redirect_ui():
    from fastapi.responses import RedirectResponse
    return RedirectResponse(url="/ui/")
```

### 9. Health Check

**Endpoint**: `GET /health` or `GET /`

**Purpose**: Simple health check

```python
@app.get("/")
@app.get("/health")
async def health_check():
    """
    Health check endpoint.
    """
    return {"status": "ok"}
```

## Middleware/Hooks System

### Hook Execution Order

In Fastify (TypeScript):
1. `preHandler` - Runs before the route handler
2. Route handler - Main endpoint logic
3. `onSend` - Runs before sending response
4. `onError` - Runs if an error occurs

### 1. Authentication Middleware

**When**: Before all requests (except public endpoints)

**Implementation**: See `08-authentication-security.md`

### 2. Agent Detection Middleware

**When**: Before `/v1/messages` requests

**Implementation**:
```python
@app.middleware("http")
async def agent_middleware(request: Request, call_next):
    """
    Detect and process agents.
    """
    if not request.url.path.startswith("/v1/messages"):
        return await call_next(request)

    if request.url.path == "/v1/messages/count_tokens":
        return await call_next(request)

    # Get request body
    body = await request.json()
    request.state.body = body

    # Check which agents should handle this request
    agents_manager = request.app.state.agents_manager
    use_agents = []

    for agent in agents_manager.get_all_agents():
        if agent.should_handle(request, request.app.state.config):
            use_agents.append(agent.name)

            # Let agent modify request
            agent.req_handler(request, request.app.state.config)

            # Inject agent tools
            if agent.tools:
                if "tools" not in body:
                    body["tools"] = []
                body["tools"].extend([
                    {
                        "name": tool.name,
                        "description": tool.description,
                        "input_schema": tool.input_schema
                    }
                    for tool in agent.tools.values()
                ])

    request.state.agents = use_agents

    return await call_next(request)
```

### 3. Router Middleware

**When**: After agent detection, before main handler

**Implementation**: See `02-routing-engine.md`

### 4. Response Processing (onSend)

**When**: Before sending response to client

**Purpose**:
- Stream response handling for agents
- Usage tracking

**Implementation**:
```python
async def on_send_hook(request: Request, response):
    """
    Process response before sending to client.
    """
    session_id = getattr(request.state, "session_id", None)

    if not session_id:
        return response

    if not request.url.path.startswith("/v1/messages"):
        return response

    if request.url.path == "/v1/messages/count_tokens":
        return response

    # Handle streaming responses
    if isinstance(response, StreamingResponse):
        # If agents are involved, process tool calls
        if request.state.agents:
            return process_agent_stream(request, response)

        # Otherwise, just track usage
        return track_usage_stream(request, response)

    # Handle non-streaming responses
    else:
        # Track usage
        if hasattr(response, "usage"):
            request.app.state.session_cache.put(
                session_id,
                Usage(
                    input_tokens=response.usage.input_tokens,
                    output_tokens=response.usage.output_tokens
                )
            )

        return response
```

### 5. Error Handling

**When**: If any error occurs

**Implementation**:
```python
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """
    Global error handler.
    """
    import traceback

    # Log error
    request.app.state.logger.error(
        f"Unhandled exception: {exc}\n{traceback.format_exc()}"
    )

    # Return error response in Anthropic format
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "type": "internal_server_error",
                "message": str(exc)
            }
        }
    )
```

## Logging Configuration

### Pino Logger (TypeScript)

The TypeScript implementation uses Pino with rotating file streams.

### Python Recommendation

```python
import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path

def setup_logging(config: Config):
    """
    Configure logging with rotation.
    """
    if not config.LOG:
        logging.disable(logging.CRITICAL)
        return

    log_dir = Path.home() / ".claude-code-router" / "logs"
    log_dir.mkdir(parents=True, exist_ok=True)

    # Create rotating file handler
    log_file = log_dir / "app.log"
    handler = RotatingFileHandler(
        log_file,
        maxBytes=50 * 1024 * 1024,  # 50MB
        backupCount=3
    )

    # Set format
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    handler.setFormatter(formatter)

    # Configure root logger
    logger = logging.getLogger()
    logger.addHandler(handler)
    logger.setLevel(getattr(logging, config.LOG_LEVEL.upper()))

    return logger
```

## Python Implementation Recommendations

### Framework Choice: FastAPI

**Why FastAPI**:
- Async/await support (required for streaming)
- Automatic request/response validation with Pydantic
- Built-in OpenAPI documentation
- Middleware system
- Dependency injection
- Great performance
- Easy to test

**Alternative: Starlette** (if you want more control):
- Lighter weight
- More explicit
- FastAPI is built on top of Starlette

### Server Structure

```python
# server.py
from fastapi import FastAPI, Request, Response
from fastapi.responses import StreamingResponse
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware
import uvicorn

from .config import Config, load_config
from .routing import RouterMiddleware
from .auth import AuthMiddleware
from .agents import AgentsManager
from .logging import setup_logging

async def create_app(config: Config) -> FastAPI:
    """Create and configure the FastAPI application."""

    app = FastAPI(
        title="Claude Code Router",
        description="Route Claude Code requests to different LLM providers",
        version="1.0.0"
    )

    # State
    app.state.config = config
    app.state.session_cache = LRUCache(100)
    app.state.agents_manager = AgentsManager()
    app.state.logger = setup_logging(config)

    # Middleware
    app.add_middleware(AuthMiddleware, config=config)
    app.add_middleware(RouterMiddleware, config=config)

    # Routes
    from .routes import messages, config as config_routes, logs, ui

    app.include_router(messages.router, prefix="/v1")
    app.include_router(config_routes.router, prefix="/api")
    app.include_router(logs.router, prefix="/api")

    # Static files
    app.mount("/ui", StaticFiles(directory="dist", html=True), name="ui")

    return app

async def start_server(config: Config):
    """Start the HTTP server."""
    app = await create_app(config)

    uvicorn.run(
        app,
        host=config.HOST,
        port=config.PORT,
        log_level=config.LOG_LEVEL.lower()
    )
```

### Testing

```python
# test_server.py
from fastapi.testclient import TestClient
import pytest

@pytest.fixture
async def client():
    config = Config(
        PORT=3456,
        Providers=[],
        Router=RouterConfig(default="test,model")
    )
    app = await create_app(config)
    return TestClient(app)

def test_health_check(client):
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}

def test_count_tokens(client):
    response = client.post("/v1/messages/count_tokens", json={
        "messages": [{"role": "user", "content": "Hello"}],
        "system": [],
        "tools": []
    })
    assert response.status_code == 200
    assert "input_tokens" in response.json()
```
