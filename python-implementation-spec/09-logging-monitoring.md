# Logging and Monitoring

## Logging System

### Two Separate Logs

1. **Server logs** (HTTP requests, errors): Rotating logs in `~/.claude-code-router/logs/ccr-*.log`
2. **Application logs** (routing decisions): `~/.claude-code-router/claude-code-router.log`

### Server Logging (Rotating)

```python
from logging.handlers import RotatingFileHandler
import logging
from pathlib import Path

def setup_server_logging(config: Config):
    """Setup rotating file logging for server."""
    if not config.LOG:
        logging.disable(logging.CRITICAL)
        return

    log_dir = Path.home() / ".claude-code-router" / "logs"
    log_dir.mkdir(parents=True, exist_ok=True)

    # Rotating handler
    log_file = log_dir / f"ccr-{datetime.now().strftime('%Y%m%d%H%M%S')}.log"
    handler = RotatingFileHandler(
        log_file,
        maxBytes=50 * 1024 * 1024,  # 50MB
        backupCount=3
    )

    # Format
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    handler.setFormatter(formatter)

    # Configure logger
    logger = logging.getLogger("ccr")
    logger.addHandler(handler)
    logger.setLevel(getattr(logging, config.LOG_LEVEL.upper()))

    return logger
```

### Log Cleanup

Keep only the 10 most recent log files:

```python
async def cleanup_log_files():
    """Clean up old log files."""
    log_dir = Path.home() / ".claude-code-router" / "logs"

    if not log_dir.exists():
        return

    log_files = sorted(
        log_dir.glob("ccr-*.log"),
        key=lambda p: p.stat().st_mtime,
        reverse=True
    )

    # Delete all but the 10 most recent
    for old_log in log_files[10:]:
        old_log.unlink()
```

### Request Logging

```python
from starlette.middleware.base import BaseHTTPMiddleware
import time

class LoggingMiddleware(BaseHTTPMiddleware):
    """Log all requests."""

    async def dispatch(self, request: Request, call_next):
        start_time = time.time()

        # Log request
        logger.info(f"{request.method} {request.url.path}")

        # Process request
        response = await call_next(request)

        # Log response
        duration = time.time() - start_time
        logger.info(
            f"{request.method} {request.url.path} - "
            f"{response.status_code} - {duration:.2f}s"
        )

        return response
```

## Status Line System

### Purpose

Display router status in Claude Code's status line (bottom of terminal).

### Configuration

```json5
{
  "StatusLine": {
    "enabled": true,
    "currentStyle": "default",  // or "powerline", "simple"
    "default": {
      "modules": [
        {
          "type": "workDir",
          "icon": "󰉋",
          "text": "{{workDirName}}",
          "color": "bright_blue"
        },
        {
          "type": "gitBranch",
          "icon": "",
          "text": "{{gitBranch}}",
          "color": "bright_magenta"
        },
        {
          "type": "model",
          "icon": "󰚩",
          "text": "{{model}}",
          "color": "bright_cyan"
        },
        {
          "type": "usage",
          "icon": "↑",
          "text": "{{inputTokens}}",
          "color": "bright_green"
        },
        {
          "type": "usage",
          "icon": "↓",
          "text": "{{outputTokens}}",
          "color": "bright_yellow"
        }
      ]
    }
  }
}
```

### Status Line Command

```bash
ccr statusline < input.json
```

**Input** (from stdin):
```json
{
  "hook_event_name": "after_request",
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/home/user/project",
  "model": {
    "id": "claude-3-5-sonnet",
    "display_name": "Claude 3.5 Sonnet"
  },
  "workspace": {
    "current_dir": "/home/user/project",
    "project_dir": "/home/user/project"
  }
}
```

**Output** (ANSI-colored string):
```
󰉋 my-project  main 󰚩 deepseek-chat ↑ 1.2k ↓ 345
```

### Implementation Overview

1. Parse input JSON
2. Read transcript file to get latest usage
3. Get git branch (if in git repo)
4. Format modules with ANSI color codes
5. Support Powerline separators (optional)
6. Output formatted string

**Note**: Full implementation is complex (700+ lines in TypeScript). See `src/utils/statusline.ts` for details.

## Monitoring Endpoints

### Health Check

```python
@app.get("/health")
async def health_check():
    return {
        "status": "ok",
        "version": "1.0.0",
        "uptime": time.time() - start_time
    }
```

### Metrics (Optional)

```python
from prometheus_client import Counter, Histogram, make_asgi_app

# Metrics
request_count = Counter('requests_total', 'Total requests', ['method', 'endpoint'])
request_duration = Histogram('request_duration_seconds', 'Request duration')

# Mount metrics endpoint
metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)
```

## Error Tracking

### Sentry Integration (Optional)

```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastAPIIntegration

sentry_sdk.init(
    dsn="your-sentry-dsn",
    integrations=[FastAPIIntegration()],
    traces_sample_rate=1.0
)
```
