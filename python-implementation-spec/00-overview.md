# Claude Code Router - Python Implementation Specification

## Document Index

This specification provides comprehensive documentation for reimplementing the Claude Code Router system in Python. The documentation is organized into the following files:

1. **00-overview.md** (this file) - System overview and architecture
2. **01-configuration-system.md** - Configuration management and data structures
3. **02-routing-engine.md** - Routing logic and model selection
4. **03-server-architecture.md** - HTTP server and API endpoints
5. **04-llm-providers-transformers.md** - Provider integration and request/response transformation
6. **05-agents-system.md** - Agent architecture and image processing
7. **06-cli-lifecycle.md** - CLI commands and process management
8. **07-streaming-sse.md** - Server-Sent Events and stream processing
9. **08-authentication-security.md** - Authentication and security features
10. **09-logging-monitoring.md** - Logging and status line system
11. **10-python-implementation.md** - Python-specific recommendations and architecture
12. **11-mlx-integration.md** - Local MLX model integration guide

## Executive Summary

Claude Code Router is a proxy service that sits between Claude Code (Anthropic's CLI tool) and various LLM providers. It intelligently routes requests to different models based on configurable rules, transforms requests/responses to match different provider APIs, and provides features like cost optimization, model switching, and extended capabilities through an agent system.

### Core Functionality

The system performs these primary functions:

1. **Request Interception**: Receives Claude Code API requests at a local endpoint
2. **Intelligent Routing**: Determines which LLM provider and model to use based on:
   - Token count (for long context)
   - Model type (background, thinking, web search, image)
   - Custom routing logic
   - Explicit model selection
3. **Request/Response Transformation**: Adapts Claude's API format to match different provider APIs
4. **Agent Processing**: Intercepts and processes special capabilities (e.g., image analysis)
5. **Response Streaming**: Streams responses back to Claude Code in real-time
6. **Session Management**: Tracks usage across conversation sessions

### Key Design Principles

- **Transparency**: Claude Code sees a standard Anthropic API endpoint
- **Flexibility**: Highly configurable routing and transformation
- **Extensibility**: Plugin system for custom transformers and routers
- **Performance**: Efficient streaming and caching
- **Reliability**: Graceful error handling and fallback routing

## System Architecture

### High-Level Architecture

```
┌─────────────────┐
│   Claude Code   │ (User's terminal)
└────────┬────────┘
         │ HTTP API calls (localhost:3456)
         │ Anthropic Message API format
         ▼
┌──────────────────────────────────────────────────┐
│         Claude Code Router Service               │
│  ┌────────────────────────────────────────────┐  │
│  │   HTTP Server (Fastify-based)              │  │
│  │   • /v1/messages (streaming)               │  │
│  │   • /v1/messages/count_tokens              │  │
│  │   • /api/config, /api/logs, /ui/*          │  │
│  └────────────┬───────────────────────────────┘  │
│               │                                   │
│  ┌────────────▼───────────────────────────────┐  │
│  │   Middleware Pipeline                      │  │
│  │   1. Authentication (API Key)              │  │
│  │   2. Agent Detection & Processing          │  │
│  │   3. Router (Model Selection)              │  │
│  └────────────┬───────────────────────────────┘  │
│               │                                   │
│  ┌────────────▼───────────────────────────────┐  │
│  │   Provider Service                         │  │
│  │   • Load transformers                      │  │
│  │   • Transform request                      │  │
│  │   • Call LLM API                          │  │
│  │   • Transform response                     │  │
│  │   • Stream back to client                 │  │
│  └────────────┬───────────────────────────────┘  │
│               │                                   │
│  ┌────────────▼───────────────────────────────┐  │
│  │   Supporting Services                      │  │
│  │   • Session Cache (LRU)                   │  │
│  │   • Image Cache (for agent)               │  │
│  │   • Configuration Management              │  │
│  │   • Logging                               │  │
│  └───────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
         │ HTTP API calls to various providers
         ▼
┌──────────────────────────────────────────────────┐
│         LLM Providers                            │
│  • OpenRouter                                    │
│  • DeepSeek                                      │
│  • Gemini                                        │
│  • Ollama (local)                                │
│  • MLX Models (local, Python-specific)           │
│  • ... many others                               │
└──────────────────────────────────────────────────┘
```

### Component Responsibilities

#### 1. CLI Module (`src/cli.ts`)
- Entry point for the `ccr` command
- Commands: start, stop, restart, status, code, model, ui, statusline
- Process lifecycle management
- Spawns Claude Code with appropriate environment variables

#### 2. Server Module (`src/server.ts`, `src/index.ts`)
- Creates and configures the HTTP server (based on Fastify)
- Registers API endpoints
- Configures middleware hooks
- Manages server lifecycle

#### 3. Configuration System (`src/utils/index.ts`, `src/constants.ts`)
- Loads and validates `~/.claude-code-router/config.json`
- Supports JSON5 format with comments
- Environment variable interpolation
- Configuration backup and updates

#### 4. Routing Engine (`src/utils/router.ts`)
- Token counting using tiktoken
- Model selection logic:
  - Default routing
  - Long context detection
  - Background task detection (Haiku models)
  - Thinking mode detection
  - Web search detection
  - Custom router support
- Session usage tracking

#### 5. Authentication (`src/middleware/auth.ts`)
- API key validation
- CORS handling for local-only access
- Access level determination

#### 6. Provider Service (from `@musistudio/llms`)
- Transformer system for request/response adaptation
- HTTP client for calling LLM APIs
- Streaming response handling
- Error handling and retries

#### 7. Agent System (`src/agents/`)
- Agent manager
- Image agent for vision capabilities
- Tool injection into requests
- Agent response processing

#### 8. Stream Processing (`src/utils/`)
- SSE parser (converts SSE to objects)
- SSE serializer (converts objects to SSE)
- Stream rewriting for agent processing
- Response tee-ing for usage tracking

#### 9. Supporting Utilities
- Process management (`src/utils/processCheck.ts`)
- Cache (LRU for session usage) (`src/utils/cache.ts`)
- Log cleanup (`src/utils/logCleanup.ts`)
- Status line formatting (`src/utils/statusline.ts`)
- Model selector UI (`src/utils/modelSelector.ts`)

## Data Flow

### Standard Request Flow

1. **Claude Code makes request** → `POST http://localhost:3456/v1/messages`
   - Request body: Anthropic Messages API format
   - Headers: `x-api-key`, `content-type: application/json`
   - May include `stream: true`

2. **Authentication middleware**
   - Validates API key (if configured)
   - Sets CORS headers (if no API key)

3. **Agent detection middleware**
   - Checks if request should be handled by an agent
   - For image agent: detects images in message history
   - Injects agent tools into request
   - Modifies request body (e.g., replaces images with placeholders)

4. **Router middleware**
   - Parses session ID from `metadata.user_id`
   - Counts tokens in request
   - Selects appropriate model:
     - Check for explicit model specification (`,` syntax)
     - Check token count for long context
     - Check for Haiku → background routing
     - Check for thinking mode
     - Check for web search
     - Apply custom router if configured
     - Fall back to default

5. **Provider service processing**
   - Loads appropriate transformers for provider/model
   - Transforms request to provider format
   - Makes HTTP call to provider
   - Transforms response back to Anthropic format
   - Streams response

6. **Agent processing (if applicable)**
   - Intercepts tool_use events in stream
   - Executes agent handlers
   - Makes recursive calls to router for agent processing
   - Injects tool results back into stream

7. **onSend hook**
   - Tees stream for usage tracking
   - Stores usage in session cache
   - Returns stream to client

8. **Claude Code receives response**
   - Streaming SSE format
   - Standard Anthropic Messages API response

### Configuration Update Flow

1. User edits `~/.claude-code-router/config.json` via:
   - Direct file editing
   - Web UI (`ccr ui`)
   - CLI tool (`ccr model`)

2. On next request or restart:
   - Config is reloaded
   - Transformers are re-initialized
   - New routing rules take effect

### Process Management Flow

1. **Start**: `ccr start`
   - Checks if already running (PID file)
   - Initializes directories
   - Loads config
   - Starts HTTP server
   - Writes PID to `~/.claude-code-router/.claude-code-router.pid`

2. **Stop**: `ccr stop`
   - Reads PID from file
   - Sends SIGTERM to process
   - Cleans up PID file

3. **Code execution**: `ccr code "prompt"`
   - Checks if service is running
   - Starts service if needed (detached process)
   - Waits for service to be ready
   - Spawns Claude Code with:
     - `ANTHROPIC_BASE_URL=http://127.0.0.1:3456`
     - `ANTHROPIC_AUTH_TOKEN=<config.APIKEY or "test">`
   - Tracks reference count
   - Cleans up on exit

## File System Layout

```
~/.claude-code-router/
├── config.json                    # Main configuration
├── config.json.*.bak             # Backup configs (keep 3 most recent)
├── .claude-code-router.pid       # Process ID file
├── logs/                         # Log directory
│   └── ccr-*.log                # Rotating logs
└── plugins/                      # Custom transformers/routers
    ├── custom-router.js
    └── my-transformer.js

~/.claude.json                     # Claude Code config (auto-created)

/tmp/claude-code-reference-count.txt  # Reference counting for auto-shutdown
```

## Key Technologies (TypeScript Implementation)

- **Runtime**: Node.js
- **Build**: esbuild
- **Server**: Fastify (via `@musistudio/llms`)
- **CLI**: child_process, minimist
- **Config**: JSON5 (JSON with comments)
- **Token counting**: tiktoken
- **Caching**: lru-cache
- **Logging**: rotating-file-stream
- **Process management**: find-process
- **Interactive prompts**: @inquirer/prompts

## External Dependencies

The system depends on:
- **Claude Code** (`@anthropic-ai/claude-code`) - installed globally
- **LLM Provider APIs** - various external services
- **User's home directory** - for config and state

## Next Steps

Continue reading the detailed specifications in the numbered documents to understand each subsystem in depth. The specifications are designed to provide all information needed to implement a clean-room Python version.
