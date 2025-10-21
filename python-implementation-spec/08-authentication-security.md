# Authentication and Security

## API Key Authentication

### Configuration

```json5
{
  "APIKEY": "your-secret-key-here"
}
```

If `APIKEY` is set, all API requests must include it.

### Header Formats

Clients can provide the API key in two ways:

1. **Authorization header** (Bearer token):
   ```
   Authorization: Bearer your-secret-key-here
   ```

2. **X-API-Key header**:
   ```
   x-api-key: your-secret-key-here
   ```

### Public Endpoints

These endpoints don't require authentication:
- `GET /`
- `GET /health`
- `GET /ui/*` (static files)

### Implementation

```python
from fastapi import Request, HTTPException, status
from starlette.middleware.base import BaseHTTPMiddleware

class AuthMiddleware(BaseHTTPMiddleware):
    """Authentication middleware."""

    def __init__(self, app, config: Config):
        super().__init__(app)
        self.config = config

    async def dispatch(self, request: Request, call_next):
        # Public endpoints
        if request.url.path in ["/", "/health"] or request.url.path.startswith("/ui"):
            return await call_next(request)

        api_key = self.config.APIKEY

        # No API key configured = local-only mode
        if not api_key:
            # Enable CORS for localhost
            allowed_origins = [
                f"http://127.0.0.1:{self.config.PORT}",
                f"http://localhost:{self.config.PORT}"
            ]

            origin = request.headers.get("origin")
            if origin and origin not in allowed_origins:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="CORS not allowed for this origin"
                )

            response = await call_next(request)
            response.headers["Access-Control-Allow-Origin"] = allowed_origins[0]
            return response

        # API key configured = require authentication
        auth_header = request.headers.get("authorization") or request.headers.get("x-api-key")

        if not auth_header:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="APIKEY is missing"
            )

        # Extract token
        if auth_header.startswith("Bearer "):
            token = auth_header[7:]
        else:
            token = auth_header

        # Validate token
        if token != api_key:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid API key"
            )

        # Proceed
        return await call_next(request)
```

## Host Binding Security

### Rule

If `APIKEY` is not set, `HOST` is forced to `127.0.0.1` to prevent unauthorized network access.

```python
def validate_host_apikey(config: Config) -> Config:
    """Enforce host security."""
    if config.HOST and config.HOST != "127.0.0.1" and not config.APIKEY:
        print("⚠️ API key is not set. HOST is forced to 127.0.0.1.")
        config.HOST = "127.0.0.1"
    return config
```

## Environment Variable Security

**Best Practice**: Don't hardcode API keys in config files. Use environment variables:

```json5
{
  "APIKEY": "$ROUTER_APIKEY",
  "Providers": [
    {
      "name": "openai",
      "api_key": "$OPENAI_API_KEY",
      // ...
    }
  ]
}
```

```bash
export ROUTER_APIKEY="my-secret-key"
export OPENAI_API_KEY="sk-..."
ccr start
```

## HTTPS/TLS

The router itself doesn't implement HTTPS. For production:

1. **Use a reverse proxy** (nginx, Caddy) with TLS termination
2. **Or use a service mesh** (Istio, Linkerd)

Example nginx config:
```nginx
server {
    listen 443 ssl;
    server_name router.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:3456;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## Rate Limiting

Not implemented in the current version. For production, use:
- nginx rate limiting
- API gateway (Kong, Tyk)
- Application-level rate limiting (slowapi for FastAPI)

Example with slowapi:
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/v1/messages")
@limiter.limit("10/minute")
async def handle_messages(request: Request):
    ...
```
