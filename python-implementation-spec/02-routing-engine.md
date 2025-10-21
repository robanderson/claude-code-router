# Routing Engine Specification

## Overview

The routing engine is the core intelligence of Claude Code Router. It determines which LLM provider and model should handle each request based on various criteria including token count, model type, request characteristics, and custom logic.

## Location in Codebase

**Primary file**: `src/utils/router.ts`

## Core Responsibilities

1. **Token Counting**: Calculate the number of tokens in a request
2. **Session Management**: Track usage across conversation sessions
3. **Model Selection**: Apply routing rules to select the appropriate model
4. **Custom Router Support**: Allow user-defined routing logic
5. **System Prompt Rewriting**: Optionally replace system prompts

## Token Counting

### Algorithm

Uses the `tiktoken` library with the `cl100k_base` encoding (same as used by GPT-4 and Claude).

### What Gets Counted

```python
def calculate_token_count(
    messages: List[MessageParam],
    system: Union[str, List[Dict[str, Any]]],
    tools: List[Tool]
) -> int:
    """
    Calculate total token count for a request.

    Args:
        messages: List of conversation messages
        system: System prompt (string or array of objects)
        tools: List of tool definitions

    Returns:
        Total estimated token count
    """
    import tiktoken

    enc = tiktoken.get_encoding("cl100k_base")
    token_count = 0

    # Count message tokens
    for message in messages:
        if isinstance(message.content, str):
            token_count += len(enc.encode(message.content))
        elif isinstance(message.content, list):
            for content_part in message.content:
                if content_part.type == "text":
                    token_count += len(enc.encode(content_part.text))
                elif content_part.type == "tool_use":
                    token_count += len(enc.encode(json.dumps(content_part.input)))
                elif content_part.type == "tool_result":
                    content_str = (
                        content_part.content
                        if isinstance(content_part.content, str)
                        else json.dumps(content_part.content)
                    )
                    token_count += len(enc.encode(content_str))

    # Count system prompt tokens
    if isinstance(system, str):
        token_count += len(enc.encode(system))
    elif isinstance(system, list):
        for item in system:
            if item.get("type") != "text":
                continue
            text = item.get("text", "")
            if isinstance(text, str):
                token_count += len(enc.encode(text))
            elif isinstance(text, list):
                for text_part in text:
                    token_count += len(enc.encode(text_part or ""))

    # Count tool definition tokens
    if tools:
        for tool in tools:
            if tool.get("description"):
                token_count += len(enc.encode(tool["name"] + tool["description"]))
            if tool.get("input_schema"):
                token_count += len(enc.encode(json.dumps(tool["input_schema"])))

    return token_count
```

### Token Count Endpoint

The router also provides an endpoint for Claude Code to estimate token counts before making requests:

```
POST /v1/messages/count_tokens

Request body:
{
  "messages": [...],
  "system": "...",
  "tools": [...]
}

Response:
{
  "input_tokens": 1234
}
```

## Session Usage Tracking

### Purpose

Track the token usage of the last response in each conversation session. This is used to:
- Detect when a session is accumulating many tokens (context growth)
- Trigger long context routing more aggressively

### Implementation

```python
from typing import Optional
from dataclasses import dataclass

@dataclass
class Usage:
    input_tokens: int
    output_tokens: int

class LRUCache:
    """
    Least Recently Used cache with fixed capacity.
    """
    def __init__(self, capacity: int = 100):
        self.capacity = capacity
        self.cache: OrderedDict[str, Usage] = OrderedDict()

    def get(self, key: str) -> Optional[Usage]:
        if key not in self.cache:
            return None
        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: str, value: Usage) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        # Remove oldest if over capacity
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)

# Global session cache
session_usage_cache = LRUCache(capacity=100)
```

### Session ID Extraction

Session IDs are embedded in the request's `metadata.user_id` field:

```python
def extract_session_id(request_body: dict) -> Optional[str]:
    """
    Extracts session ID from metadata.user_id field.

    Format: "user_id_session_SESSION_ID"

    Example:
        user_id = "abc123_session_xyz789"
        returns: "xyz789"
    """
    user_id = request_body.get("metadata", {}).get("user_id")
    if not user_id:
        return None

    parts = user_id.split("_session_")
    if len(parts) > 1:
        return parts[1]

    return None
```

## Model Selection Logic

### Selection Flow

```python
async def get_use_model(
    request: Request,
    token_count: int,
    config: Config,
    last_usage: Optional[Usage] = None
) -> str:
    """
    Determines which model to use for this request.

    Priority order:
    1. Explicit model specification (provider,model format)
    2. Subagent model specification
    3. Long context threshold
    4. Background model (Haiku detection)
    5. Web search model
    6. Thinking model
    7. Default model

    Returns:
        Model string in format "provider,model"
    """
    body = request.body

    # 1. Check for explicit model specification
    if "," in body.get("model", ""):
        provider, model = body["model"].split(",", 1)
        # Validate provider and model exist
        final_provider = next(
            (p for p in config.Providers if p.name.lower() == provider.lower()),
            None
        )
        if final_provider:
            final_model = next(
                (m for m in final_provider.models if m.lower() == model.lower()),
                None
            )
            if final_model:
                return f"{final_provider.name},{final_model}"

    # 2. Check for subagent model specification in system prompt
    system = body.get("system", [])
    if isinstance(system, list) and len(system) > 1:
        system_text = system[1].get("text", "")
        if system_text.startswith("<CCR-SUBAGENT-MODEL>"):
            import re
            match = re.search(
                r'<CCR-SUBAGENT-MODEL>(.*?)</CCR-SUBAGENT-MODEL>',
                system_text,
                re.DOTALL
            )
            if match:
                model = match.group(1)
                # Remove the tag from the system prompt
                system[1]["text"] = system_text.replace(
                    f"<CCR-SUBAGENT-MODEL>{model}</CCR-SUBAGENT-MODEL>",
                    ""
                )
                return model

    # 3. Check for long context
    long_context_threshold = config.Router.longContextThreshold or 60000

    # Check if last usage was over threshold and current is also large
    last_usage_threshold = (
        last_usage and
        last_usage.input_tokens > long_context_threshold and
        token_count > 20000
    )

    # Or check if current request alone exceeds threshold
    token_count_threshold = token_count > long_context_threshold

    if (last_usage_threshold or token_count_threshold) and config.Router.longContext:
        request.log.info(
            f"Using long context model due to token count: {token_count}, "
            f"threshold: {long_context_threshold}"
        )
        return config.Router.longContext

    # 4. Check for Haiku background model
    # Any Claude Haiku variant routes to background model
    model = body.get("model", "")
    if ("claude" in model.lower() and
        "haiku" in model.lower() and
        config.Router.background):
        request.log.info(f"Using background model for {model}")
        return config.Router.background

    # 5. Check for web search
    # Priority of webSearch is higher than thinking
    tools = body.get("tools", [])
    if tools:
        has_web_search = any(
            tool.get("type", "").startswith("web_search")
            for tool in tools
        )
        if has_web_search and config.Router.webSearch:
            return config.Router.webSearch

    # 6. Check for thinking mode
    if body.get("thinking") and config.Router.think:
        request.log.info(f"Using think model for {body['thinking']}")
        return config.Router.think

    # 7. Default model
    return config.Router.default
```

### Routing Rules Detailed

#### 1. Explicit Model Specification

**Format**: `provider,model`

**Usage**: User can specify model in Claude Code using:
```
/model openrouter,anthropic/claude-3.5-sonnet
```

This sets `request.body.model = "openrouter,anthropic/claude-3.5-sonnet"`

**Validation**:
- Provider name must match a configured provider (case-insensitive)
- Model name must exist in that provider's model list (case-insensitive)
- If validation fails, fall through to other routing logic

#### 2. Subagent Model Specification

**Format**: Special tag in system prompt

```
<CCR-SUBAGENT-MODEL>provider,model</CCR-SUBAGENT-MODEL>
```

**Location**: Must be at the beginning of `system[1].text`

**Purpose**: Allows Claude Code's agent system to specify which model should handle a subagent task

**Processing**:
- Extract model from tag
- Remove tag from system prompt (to avoid confusing the LLM)
- Return extracted model

#### 3. Long Context Detection

**Triggers**:
- **Current request token count** > `longContextThreshold` (default: 60000)
- **OR Previous response** had > `longContextThreshold` input tokens **AND current request** > 20000 tokens

**Rationale**:
- Long conversations accumulate context over time
- Once context is large, it tends to stay large
- Using both conditions allows early switching to long-context models

**Example**:
```
Request 1: 5000 tokens → default model
Response 1: input_tokens = 5000
Request 2: 10000 tokens → default model
Response 2: input_tokens = 10000
...
Request 10: 65000 tokens → longContext model
Response 10: input_tokens = 65000
Request 11: 25000 tokens → longContext model (because previous was > threshold)
```

#### 4. Background Model (Haiku Detection)

**Trigger**: Request model contains both "claude" and "haiku" (case-insensitive)

**Purpose**: Claude Code uses Haiku models for background tasks (like file searching, grepping). These tasks are:
- Frequent
- Low complexity
- Cost-sensitive

**Strategy**: Route to a cheaper/local model

**Example**:
```python
model = "claude-3-5-haiku-20241022"
if "claude" in model and "haiku" in model:
    # Route to local Ollama or cheap provider
    return "ollama,qwen2.5-coder:latest"
```

#### 5. Web Search Model

**Trigger**: Request includes a tool with `type` starting with `"web_search"`

**Purpose**: Web search requires:
- Internet connectivity (some providers/models may not support it)
- Specific model capabilities
- Often needs to be routed to providers that support the feature

**Priority**: Higher than thinking mode

**Example**:
```python
tools = [
    {
        "type": "web_search_20250125",
        "name": "WebSearch",
        ...
    }
]
# Route to gemini,gemini-2.5-flash (supports web search)
```

#### 6. Thinking Model

**Trigger**: Request has `thinking` field set to truthy value

**Purpose**: Claude Code's "Plan Mode" and reasoning tasks set this flag

**Example**:
```python
request.body.thinking = {"enabled": True}
# Route to deepseek,deepseek-reasoner
```

#### 7. Default Model

**Fallback**: If no other routing rule matches

## Custom Router Support

### Purpose

Allow users to implement complex, custom routing logic beyond the built-in rules.

### Configuration

```json5
{
  "CUSTOM_ROUTER_PATH": "/path/to/custom-router.js"
}
```

### Custom Router Interface

The custom router is a JavaScript/Python module that exports an async function:

```javascript
// custom-router.js
/**
 * @param {object} req - Request object with body, headers, etc.
 * @param {object} config - Application configuration
 * @param {object} context - Additional context (event emitter, etc.)
 * @returns {Promise<string|null>} - "provider,model" or null to use default routing
 */
module.exports = async function router(req, config, context) {
    // Access request body
    const messages = req.body.messages;
    const system = req.body.system;

    // Custom logic examples:

    // Route based on message content
    const userMessage = messages.find(m => m.role === "user")?.content;
    if (userMessage && userMessage.includes("explain")) {
        return "openrouter,anthropic/claude-3.5-sonnet";
    }

    // Route based on time of day
    const hour = new Date().getHours();
    if (hour >= 22 || hour <= 6) {  // Night time
        return "ollama,local-model";  // Use local model to save costs
    }

    // Route based on token count (available as req.tokenCount)
    if (req.tokenCount > 100000) {
        return "gemini,gemini-2.5-pro";
    }

    // Route based on system prompt content
    if (system.some(s => s.text?.includes("code review"))) {
        return "deepseek,deepseek-chat";
    }

    // Return null to fall back to default routing logic
    return null;
};
```

### Python Implementation

For Python implementation, custom routers can be Python modules:

```python
# custom_router.py
from typing import Optional, Dict, Any

async def router(
    req: Request,
    config: Config,
    context: Dict[str, Any]
) -> Optional[str]:
    """
    Custom routing logic.

    Args:
        req: Request object with body, headers, session_id, etc.
        config: Application configuration
        context: Additional context (event emitter, etc.)

    Returns:
        "provider,model" string or None for default routing
    """
    body = req.body

    # Your custom logic here
    messages = body.get("messages", [])
    user_message = next(
        (m["content"] for m in messages if m["role"] == "user"),
        None
    )

    if user_message and "explain" in user_message:
        return "openrouter,anthropic/claude-3.5-sonnet"

    # Fallback to default
    return None
```

### Loading and Executing Custom Router

```python
import importlib.util
from pathlib import Path

async def load_custom_router(router_path: str):
    """
    Dynamically load a custom router module.
    """
    path = Path(router_path)
    if not path.exists():
        raise FileNotFoundError(f"Custom router not found: {router_path}")

    spec = importlib.util.spec_from_file_location("custom_router", path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)

    if not hasattr(module, "router"):
        raise AttributeError("Custom router must export a 'router' function")

    return module.router

async def apply_custom_router(
    req: Request,
    config: Config,
    context: dict
) -> Optional[str]:
    """
    Apply custom router if configured.
    """
    if not config.CUSTOM_ROUTER_PATH:
        return None

    try:
        custom_router = await load_custom_router(config.CUSTOM_ROUTER_PATH)
        model = await custom_router(req, config, context)
        return model
    except Exception as e:
        req.log.error(f"Failed to load/execute custom router: {e}")
        return None
```

## System Prompt Rewriting

### Purpose

Allow users to replace the system prompt that Claude Code sends.

### Configuration

```json5
{
  "REWRITE_SYSTEM_PROMPT": "/path/to/custom-prompt.txt"
}
```

### Implementation

```python
async def rewrite_system_prompt(
    system: List[Dict[str, Any]],
    config: Config
) -> List[Dict[str, Any]]:
    """
    Optionally rewrite the system prompt.

    The system prompt from Claude Code has a specific structure:
    [
        { "type": "text", "text": "main instructions..." },
        { "type": "text", "text": "<env>environment info...</env>..." }
    ]

    We replace the main instructions but preserve the environment info.
    """
    if not config.REWRITE_SYSTEM_PROMPT:
        return system

    if len(system) < 2:
        return system

    if "<env>" not in system[1].get("text", ""):
        return system

    # Read custom prompt
    prompt_path = Path(config.REWRITE_SYSTEM_PROMPT)
    if not prompt_path.exists():
        return system

    custom_prompt = prompt_path.read_text()

    # Extract environment section from original
    original_text = system[1]["text"]
    env_section = original_text.split("<env>", 1)[1]  # Everything after <env>

    # Combine custom prompt with env section
    system[1]["text"] = f"{custom_prompt}<env>{env_section}"

    return system
```

## Router Middleware Integration

### When Router Runs

The router runs as a pre-handler middleware for `/v1/messages` requests:

```python
@app.before_request("/v1/messages")
async def router_middleware(request: Request):
    """
    Router middleware that runs before the main handler.
    """
    # Skip for count_tokens endpoint
    if request.path == "/v1/messages/count_tokens":
        return

    # Extract session ID
    session_id = extract_session_id(request.body)
    request.session_id = session_id

    # Get last usage for this session
    last_usage = None
    if session_id:
        last_usage = session_usage_cache.get(session_id)

    # Count tokens
    token_count = calculate_token_count(
        request.body.get("messages", []),
        request.body.get("system", []),
        request.body.get("tools", [])
    )
    request.token_count = token_count

    # Rewrite system prompt if configured
    if config.REWRITE_SYSTEM_PROMPT:
        request.body["system"] = await rewrite_system_prompt(
            request.body.get("system", []),
            config
        )

    # Try custom router first
    model = None
    if config.CUSTOM_ROUTER_PATH:
        try:
            custom_router = await load_custom_router(config.CUSTOM_ROUTER_PATH)
            model = await custom_router(request, config, {"event": event_emitter})
        except Exception as e:
            request.log.error(f"Custom router failed: {e}")

    # Fall back to built-in routing
    if not model:
        model = await get_use_model(request, token_count, config, last_usage)

    # Set the model on the request
    request.body["model"] = model
```

## Python Implementation Recommendations

### Libraries

- **tiktoken**: Token counting (official OpenAI library, works for Claude too)
  ```bash
  pip install tiktoken
  ```

- **collections.OrderedDict**: For LRU cache implementation (standard library)

### Code Organization

```
routing/
├── __init__.py
├── token_counter.py      # Token counting logic
├── session_cache.py      # LRU cache for session usage
├── model_selector.py     # Model selection logic
├── custom_router.py      # Custom router loader
└── middleware.py         # Fastify/Starlette middleware integration
```

### Testing Considerations

Test cases should cover:
- Token counting for different message types
- Long context detection (both threshold conditions)
- Haiku detection (various model name formats)
- Web search detection
- Thinking mode detection
- Explicit model specification
- Subagent model extraction
- Custom router execution
- Fallback to default
- Session cache LRU behavior
