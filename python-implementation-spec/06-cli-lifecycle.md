# CLI and Server Lifecycle Management

## CLI Commands

The `ccr` command provides the following subcommands:

### 1. start - Start Server

```bash
ccr start
```

**Behavior**:
- Check if service is already running (via PID file)
- If running, print message and exit
- Initialize directories (`~/.claude-code-router`, `logs`, `plugins`)
- Initialize `~/.claude.json` for Claude Code
- Load configuration
- Clean up old log files (keep 10 most recent)
- Start HTTP server
- Save PID to file
- Run in foreground

**Implementation**:
```python
async def cmd_start():
    """Start the router service."""
    # Check if already running
    if await is_service_running():
        print("✅ Service is already running in the background.")
        return

    # Initialize
    await initialize_claude_config()
    await init_dir()
    await cleanup_log_files()

    # Load config
    config = await init_config()

    # Save PID
    save_pid(os.getpid())

    # Setup signal handlers
    signal.signal(signal.SIGINT, lambda s, f: cleanup_and_exit())
    signal.signal(signal.SIGTERM, lambda s, f: cleanup_and_exit())

    # Start server
    await start_server(config)
```

### 2. stop - Stop Server

```bash
ccr stop
```

**Behavior**:
- Read PID from file
- Send SIGTERM to process
- Clean up PID file
- Clean up reference count file

**Implementation**:
```python
def cmd_stop():
    """Stop the router service."""
    try:
        pid = int(Path(PID_FILE).read_text())
        os.kill(pid, signal.SIGTERM)
        cleanup_pid_file()
        if Path(REFERENCE_COUNT_FILE).exists():
            Path(REFERENCE_COUNT_FILE).unlink()
        print("claude code router service has been successfully stopped.")
    except (FileNotFoundError, ProcessLookupError):
        print("Failed to stop the service. It may have already been stopped.")
        cleanup_pid_file()
```

### 3. restart - Restart Server

```bash
ccr restart
```

**Behavior**:
- Stop service (if running)
- Start service in background (detached)

**Implementation**:
```python
def cmd_restart():
    """Restart the router service."""
    # Stop
    cmd_stop()

    # Start in background
    print("Starting claude code router service...")
    subprocess.Popen(
        [sys.executable, "-m", "claude_code_router", "start"],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
        start_new_session=True  # Detach from terminal
    )
    print("✅ Service started successfully in the background.")
```

### 4. status - Show Service Status

```bash
ccr status
```

**Behavior**:
- Check if service is running
- Show PID, port, endpoint
- Show reference count (active Claude Code instances)

**Implementation**:
```python
async def cmd_status():
    """Show service status."""
    info = await get_service_info()

    print(f"Service Status:")
    print(f"  Running: {info['running']}")
    if info['running']:
        print(f"  PID: {info['pid']}")
        print(f"  Port: {info['port']}")
        print(f"  Endpoint: {info['endpoint']}")
        print(f"  Active connections: {info['referenceCount']}")
```

### 5. code - Execute Claude Code

```bash
ccr code "Write a function to add two numbers"
ccr code --model provider,model "Hello"
```

**Behavior**:
- Check if service is running
- If not, start it (detached) and wait for it to be ready
- Spawn Claude Code with environment variables:
  - `ANTHROPIC_BASE_URL=http://127.0.0.1:3456`
  - `ANTHROPIC_AUTH_TOKEN=<APIKEY or "test">`
  - `NO_PROXY=127.0.0.1`
  - `DISABLE_TELEMETRY=true`
  - `DISABLE_COST_WARNINGS=true`
- Increment reference count
- Wait for Claude Code to complete
- Decrement reference count
- Optionally shut down service if reference count reaches 0

**Implementation**:
```python
async def cmd_code(args: list[str]):
    """Execute Claude Code through the router."""
    # Check if service is running
    if not await is_service_running():
        print("Service not running, starting service...")

        # Start service in background
        subprocess.Popen(
            [sys.executable, "-m", "claude_code_router", "start"],
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
            start_new_session=True
        )

        # Wait for service to be ready
        if not await wait_for_service(timeout=10000):
            print("Service startup timeout. Please manually run `ccr start`.")
            sys.exit(1)

    # Execute Claude Code
    await execute_code_command(args)


async def execute_code_command(args: list[str]):
    """Execute Claude Code with proper environment."""
    config = await read_config_file()
    port = config.PORT or 3456

    # Setup environment
    env = os.environ.copy()
    env.update({
        "ANTHROPIC_AUTH_TOKEN": config.APIKEY or "test",
        "ANTHROPIC_API_KEY": "",
        "ANTHROPIC_BASE_URL": f"http://127.0.0.1:{port}",
        "NO_PROXY": "127.0.0.1",
        "DISABLE_TELEMETRY": "true",
        "DISABLE_COST_WARNINGS": "true",
        "API_TIMEOUT_MS": str(config.API_TIMEOUT_MS or 600000)
    })

    # Add statusline if configured
    if config.get("StatusLine", {}).get("enabled"):
        settings = {
            "env": {k: v for k, v in env.items()},
            "statusLine": {
                "type": "command",
                "command": "ccr statusline",
                "padding": 0
            }
        }
        args.extend(["--settings", json.dumps(settings)])

    # Non-interactive mode
    if config.NON_INTERACTIVE_MODE:
        env.update({
            "CI": "true",
            "FORCE_COLOR": "0",
            "NODE_NO_READLINE": "1",
            "TERM": "dumb"
        })

    # Increment reference count
    increment_reference_count()

    # Execute Claude Code
    claude_path = config.CLAUDE_PATH or "claude"

    try:
        process = subprocess.Popen(
            [claude_path] + args,
            env=env,
            stdin=subprocess.PIPE if config.NON_INTERACTIVE_MODE else None
        )

        # Close stdin for non-interactive mode
        if config.NON_INTERACTIVE_MODE and process.stdin:
            process.stdin.close()

        # Wait for completion
        exit_code = process.wait()

    except Exception as e:
        print(f"Failed to start claude command: {e}")
        print("Make sure Claude Code is installed: npm install -g @anthropic-ai/claude-code")
        exit_code = 1

    finally:
        # Decrement reference count
        decrement_reference_count()

        # Optionally close service if no more references
        close_service()

    sys.exit(exit_code)
```

### 6. model - Interactive Model Selector

```bash
ccr model
```

**Behavior**:
- Load current configuration
- Display current model assignments
- Interactive prompts to:
  - Select model type to update (default, background, think, etc.)
  - Select from available models
  - Or add new model/provider
- Save updated configuration

**Implementation**: See `src/utils/modelSelector.ts` for full details. Python can use `prompt_toolkit` or `questionary` for interactive prompts.

### 7. ui - Open Web UI

```bash
ccr ui
```

**Behavior**:
- Check if service is running
- If not, start it
- Open browser to `http://127.0.0.1:{PORT}/ui/`

**Implementation**:
```python
async def cmd_ui():
    """Open the web UI."""
    # Ensure service is running
    if not await is_service_running():
        print("Service not running, starting service...")
        # ... start service

    # Get service info
    info = await get_service_info()
    url = f"{info['endpoint']}/ui/"

    print(f"Opening UI at {url}")

    # Open browser
    import webbrowser
    webbrowser.open(url)
```

### 8. statusline - Status Line Formatter

```bash
echo '{"hook_event_name": "..."}' | ccr statusline
```

**Behavior**:
- Read JSON from stdin
- Parse session data
- Format and output status line

**Implementation**: See `src/utils/statusline.ts`. This is complex with ANSI color codes and Nerd Fonts support.

## Process Management

### PID File Management

**Location**: `~/.claude-code-router/.claude-code-router.pid`

```python
PID_FILE = Path.home() / ".claude-code-router" / ".claude-code-router.pid"

def save_pid(pid: int):
    """Save process ID to file."""
    PID_FILE.write_text(str(pid))

def get_service_pid() -> Optional[int]:
    """Get saved PID."""
    if not PID_FILE.exists():
        return None
    try:
        return int(PID_FILE.read_text())
    except ValueError:
        return None

def cleanup_pid_file():
    """Remove PID file."""
    if PID_FILE.exists():
        PID_FILE.unlink()
```

### Process Checking

```python
async def is_service_running() -> bool:
    """Check if service is running."""
    pid = get_service_pid()
    if not pid:
        return False

    try:
        # Check if process exists (signal 0 doesn't kill)
        os.kill(pid, 0)
        return True
    except OSError:
        # Process doesn't exist
        cleanup_pid_file()
        return False
```

### Reference Counting

**Purpose**: Track how many Claude Code instances are using the service. Allows auto-shutdown when all clients disconnect.

**Location**: `/tmp/claude-code-reference-count.txt`

```python
REFERENCE_COUNT_FILE = Path("/tmp/claude-code-reference-count.txt")

def increment_reference_count():
    """Increment reference count."""
    count = get_reference_count()
    REFERENCE_COUNT_FILE.write_text(str(count + 1))

def decrement_reference_count():
    """Decrement reference count."""
    count = max(0, get_reference_count() - 1)
    REFERENCE_COUNT_FILE.write_text(str(count))

def get_reference_count() -> int:
    """Get current reference count."""
    if not REFERENCE_COUNT_FILE.exists():
        return 0
    try:
        return int(REFERENCE_COUNT_FILE.read_text())
    except ValueError:
        return 0

def close_service():
    """Close service if reference count is 0."""
    if get_reference_count() == 0:
        # Optionally shut down service
        # (Current implementation doesn't auto-shutdown)
        pass
```

### Wait for Service

```python
async def wait_for_service(timeout: int = 10000, initial_delay: int = 1000) -> bool:
    """
    Wait for service to become ready.

    Args:
        timeout: Total timeout in milliseconds
        initial_delay: Initial delay before first check in milliseconds

    Returns:
        True if service is ready, False if timeout
    """
    import asyncio

    # Initial delay
    await asyncio.sleep(initial_delay / 1000)

    # Poll until ready or timeout
    start_time = time.time() * 1000
    while (time.time() * 1000 - start_time) < timeout:
        if await is_service_running():
            # Additional delay to ensure fully ready
            await asyncio.sleep(0.5)
            return True
        await asyncio.sleep(0.1)

    return False
```

## Python CLI Implementation

### Recommended Library: click

```python
import click

@click.group()
def cli():
    """Claude Code Router CLI"""
    pass

@cli.command()
def start():
    """Start the router service"""
    asyncio.run(cmd_start())

@cli.command()
def stop():
    """Stop the router service"""
    cmd_stop()

@cli.command()
@click.argument('args', nargs=-1)
def code(args):
    """Execute Claude Code through the router"""
    asyncio.run(cmd_code(list(args)))

@cli.command()
def model():
    """Interactive model selector"""
    asyncio.run(cmd_model())

# ... more commands

if __name__ == "__main__":
    cli()
```

### Entry Point (pyproject.toml)

```toml
[project.scripts]
ccr = "claude_code_router.cli:cli"
```
