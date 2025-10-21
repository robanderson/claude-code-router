# MLX Local Model Integration Guide

## Overview

This guide explains how to integrate Apple MLX-based local models into the Python implementation of Claude Code Router. MLX is Apple's machine learning framework optimized for Apple Silicon (M1/M2/M3/M4 Macs).

## Why Python + MLX?

- **Python is required for MLX**: MLX has Python bindings (and C++ but Python is primary)
- **Native performance**: MLX leverages Apple Silicon's unified memory architecture
- **Model availability**: Many quantized models available for MLX (Qwen, Llama, Mistral, etc.)
- **Local inference**: No API calls, complete privacy, no costs

## MLX Setup

### Installation

```bash
# Install MLX and MLX-LM (language models)
pip install mlx>=0.4.0
pip install mlx-lm>=0.4.0

# Optional: Install helpers
pip install huggingface_hub  # For downloading models
```

### Downloading Models

```bash
# Example: Download a quantized model
python -c "from huggingface_hub import snapshot_download; snapshot_download(repo_id='mlx-community/Qwen2.5-Coder-7B-Instruct-4bit', local_dir='models/qwen2.5-coder-7b')"
```

**Popular MLX models**:
- `mlx-community/Qwen2.5-Coder-7B-Instruct-4bit` - Excellent for coding
- `mlx-community/Llama-3.2-3B-Instruct-4bit` - Fast, lightweight
- `mlx-community/Meta-Llama-3.1-8B-Instruct-4bit` - Good general purpose
- `mlx-community/DeepSeek-R1-Distill-Qwen-7B-4bit` - Reasoning model

## MLX Transformer Implementation

### MLX Provider Configuration

```json5
{
  "Providers": [
    {
      "name": "mlx",
      "api_base_url": "http://127.0.0.1:3457/v1/chat/completions",  // Internal MLX server
      "api_key": "mlx-local",
      "models": [
        "qwen2.5-coder-7b",
        "llama-3.1-8b",
        "deepseek-r1-distill-7b"
      ],
      "transformer": {
        "use": ["mlx"]
      }
    }
  ],
  "Router": {
    "default": "mlx,qwen2.5-coder-7b",
    "background": "mlx,qwen2.5-coder-7b",  // Use local model for background tasks
    "think": "openrouter,claude-3.5-sonnet"  // Use cloud for complex reasoning
  },
  "MLX": {
    "models_dir": "~/.claude-code-router/models",
    "max_tokens": 4096,
    "temperature": 0.7
  }
}
```

### MLX Transformer Class

```python
# src/claude_code_router/providers/transformers/mlx.py

from typing import Dict, Any, AsyncIterator
from ..base import Transformer

class MLXTransformer(Transformer):
    """
    Transformer for MLX local models.

    MLX models use OpenAI-compatible format but may need adjustments.
    """

    async def transform_request(self, request: dict) -> dict:
        """
        Transform Anthropic format to OpenAI format (MLX compatible).
        """
        # Extract system prompt
        system_prompt = ""
        if "system" in request:
            if isinstance(request["system"], str):
                system_prompt = request["system"]
            elif isinstance(request["system"], list):
                system_prompt = "\n".join(
                    item.get("text", "")
                    for item in request["system"]
                    if item.get("type") == "text"
                )

        # Build messages
        messages = []
        if system_prompt:
            messages.append({
                "role": "system",
                "content": system_prompt
            })

        # Convert Anthropic messages to OpenAI format
        for msg in request.get("messages", []):
            role = msg["role"]
            content = msg["content"]

            # Handle string content
            if isinstance(content, str):
                messages.append({"role": role, "content": content})
                continue

            # Handle complex content (list of blocks)
            text_parts = []
            for block in content:
                if block.get("type") == "text":
                    text_parts.append(block.get("text", ""))
                elif block.get("type") == "tool_result":
                    # Include tool results as text
                    result = block.get("content", "")
                    if isinstance(result, str):
                        text_parts.append(f"[Tool Result] {result}")

            if text_parts:
                messages.append({
                    "role": role,
                    "content": "\n".join(text_parts)
                })

        return {
            "model": request.get("model"),
            "messages": messages,
            "max_tokens": request.get("max_tokens", 4096),
            "temperature": request.get("temperature", 0.7),
            "stream": request.get("stream", False)
        }

    async def transform_response(self, response: dict) -> dict:
        """
        Transform OpenAI format response to Anthropic format.
        """
        choice = response["choices"][0]
        message = choice["message"]

        return {
            "id": response.get("id", "msg_" + uuid.uuid4().hex[:8]),
            "type": "message",
            "role": "assistant",
            "content": [{
                "type": "text",
                "text": message.get("content", "")
            }],
            "model": response.get("model"),
            "stop_reason": "end_turn" if choice.get("finish_reason") == "stop" else choice.get("finish_reason"),
            "usage": {
                "input_tokens": response.get("usage", {}).get("prompt_tokens", 0),
                "output_tokens": response.get("usage", {}).get("completion_tokens", 0)
            }
        }

    async def transform_stream(self, stream: AsyncIterator) -> AsyncIterator:
        """
        Transform OpenAI streaming format to Anthropic SSE format.
        """
        # Parse SSE stream from MLX
        parser = SSEParser()
        events = parser.parse_stream(stream)

        # Track state
        message_id = "msg_" + uuid.uuid4().hex[:8]
        input_tokens = 0  # MLX doesn't provide this in stream, estimate or skip

        # Send message_start
        yield {
            "event": "message_start",
            "data": {
                "type": "message_start",
                "message": {
                    "id": message_id,
                    "type": "message",
                    "role": "assistant",
                    "content": [],
                    "model": "mlx-model",
                    "usage": {"input_tokens": input_tokens, "output_tokens": 0}
                }
            }
        }

        # Send content_block_start
        yield {
            "event": "content_block_start",
            "data": {
                "type": "content_block_start",
                "index": 0,
                "content_block": {"type": "text", "text": ""}
            }
        }

        # Stream deltas
        total_output_tokens = 0
        async for event in events:
            if event.get("data", {}).get("choices"):
                delta = event["data"]["choices"][0].get("delta", {})
                content = delta.get("content", "")

                if content:
                    yield {
                        "event": "content_block_delta",
                        "data": {
                            "type": "content_block_delta",
                            "index": 0,
                            "delta": {"type": "text_delta", "text": content}
                        }
                    }
                    total_output_tokens += len(content.split())  # Rough estimate

        # Send content_block_stop
        yield {
            "event": "content_block_stop",
            "data": {"type": "content_block_stop", "index": 0}
        }

        # Send message_delta
        yield {
            "event": "message_delta",
            "data": {
                "type": "message_delta",
                "delta": {"stop_reason": "end_turn"},
                "usage": {"output_tokens": total_output_tokens}
            }
        }

        # Send message_stop
        yield {
            "event": "message_stop",
            "data": {"type": "message_stop"}
        }
```

## MLX Model Server

### Internal Server Implementation

Create a lightweight FastAPI server that runs MLX models:

```python
# src/claude_code_router/mlx/server.py

from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
import mlx.core as mx
import mlx_lm
from pathlib import Path
import asyncio

class MLXModelServer:
    """Internal server for MLX model inference."""

    def __init__(self, config: dict):
        self.config = config
        self.models_dir = Path(config.get("models_dir", "~/.claude-code-router/models")).expanduser()
        self.loaded_models = {}  # Cache loaded models

    def load_model(self, model_name: str):
        """Load an MLX model."""
        if model_name in self.loaded_models:
            return self.loaded_models[model_name]

        model_path = self.models_dir / model_name

        if not model_path.exists():
            raise ValueError(f"Model not found: {model_path}")

        # Load model and tokenizer
        model, tokenizer = mlx_lm.load(str(model_path))
        self.loaded_models[model_name] = (model, tokenizer)

        return model, tokenizer

    async def generate(
        self,
        model_name: str,
        messages: list,
        max_tokens: int = 4096,
        temperature: float = 0.7,
        stream: bool = False
    ):
        """Generate response from MLX model."""
        model, tokenizer = self.load_model(model_name)

        # Format prompt
        prompt = self._format_messages(messages, tokenizer)

        if stream:
            # Streaming generation
            async def generate_stream():
                for token in mlx_lm.generate(
                    model,
                    tokenizer,
                    prompt=prompt,
                    max_tokens=max_tokens,
                    temp=temperature,
                    verbose=False
                ):
                    # Yield in OpenAI streaming format
                    yield {
                        "id": "mlx_" + uuid.uuid4().hex[:8],
                        "object": "chat.completion.chunk",
                        "created": int(time.time()),
                        "model": model_name,
                        "choices": [{
                            "index": 0,
                            "delta": {"content": token},
                            "finish_reason": None
                        }]
                    }

                # Final chunk
                yield {
                    "id": "mlx_" + uuid.uuid4().hex[:8],
                    "object": "chat.completion.chunk",
                    "created": int(time.time()),
                    "model": model_name,
                    "choices": [{
                        "index": 0,
                        "delta": {},
                        "finish_reason": "stop"
                    }]
                }

            return generate_stream()

        else:
            # Non-streaming generation
            response = mlx_lm.generate(
                model,
                tokenizer,
                prompt=prompt,
                max_tokens=max_tokens,
                temp=temperature,
                verbose=False
            )

            return {
                "id": "mlx_" + uuid.uuid4().hex[:8],
                "object": "chat.completion",
                "created": int(time.time()),
                "model": model_name,
                "choices": [{
                    "index": 0,
                    "message": {
                        "role": "assistant",
                        "content": response
                    },
                    "finish_reason": "stop"
                }],
                "usage": {
                    "prompt_tokens": len(tokenizer.encode(prompt)),
                    "completion_tokens": len(tokenizer.encode(response)),
                    "total_tokens": len(tokenizer.encode(prompt)) + len(tokenizer.encode(response))
                }
            }

    def _format_messages(self, messages: list, tokenizer) -> str:
        """Format messages according to model's chat template."""
        # Most models use a chat template
        if hasattr(tokenizer, "apply_chat_template"):
            return tokenizer.apply_chat_template(messages, tokenize=False)

        # Fallback: simple concatenation
        formatted = ""
        for msg in messages:
            role = msg["role"]
            content = msg["content"]
            if role == "system":
                formatted += f"<|system|>\n{content}\n"
            elif role == "user":
                formatted += f"<|user|>\n{content}\n"
            elif role == "assistant":
                formatted += f"<|assistant|>\n{content}\n"

        formatted += "<|assistant|>\n"
        return formatted


def create_mlx_app(config: dict) -> FastAPI:
    """Create FastAPI app for MLX inference."""
    app = FastAPI(title="MLX Model Server")
    server = MLXModelServer(config)

    @app.post("/v1/chat/completions")
    async def chat_completions(request: dict):
        """OpenAI-compatible chat completions endpoint."""
        model = request.get("model")
        messages = request.get("messages", [])
        max_tokens = request.get("max_tokens", 4096)
        temperature = request.get("temperature", 0.7)
        stream = request.get("stream", False)

        try:
            result = await server.generate(
                model_name=model,
                messages=messages,
                max_tokens=max_tokens,
                temperature=temperature,
                stream=stream
            )

            if stream:
                async def stream_response():
                    async for chunk in result:
                        yield f"data: {json.dumps(chunk)}\n\n"
                    yield "data: [DONE]\n\n"

                return StreamingResponse(stream_response(), media_type="text/event-stream")
            else:
                return result

        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))

    return app


async def start_mlx_server(config: dict):
    """Start MLX model server on a different port."""
    app = create_mlx_app(config)

    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=3457, log_level="error")
```

### Starting MLX Server with Main Server

```python
# In main server startup

async def create_app() -> FastAPI:
    # ... existing code ...

    # Start MLX server if configured
    if config.get("MLX"):
        from .mlx import start_mlx_server

        # Run MLX server in background
        asyncio.create_task(start_mlx_server(config.MLX))

    return app
```

## Configuration Examples

### Example 1: Local for Background, Cloud for Main

```json5
{
  "Providers": [
    {
      "name": "mlx",
      "api_base_url": "http://127.0.0.1:3457/v1/chat/completions",
      "api_key": "mlx",
      "models": ["qwen2.5-coder-7b"],
      "transformer": {"use": ["mlx"]}
    },
    {
      "name": "openrouter",
      "api_base_url": "https://openrouter.ai/api/v1/chat/completions",
      "api_key": "$OPENROUTER_API_KEY",
      "models": ["anthropic/claude-3.5-sonnet"],
      "transformer": {"use": ["openrouter"]}
    }
  ],
  "Router": {
    "default": "openrouter,anthropic/claude-3.5-sonnet",
    "background": "mlx,qwen2.5-coder-7b",  // Fast local model for background
    "longContext": "openrouter,anthropic/claude-3.5-sonnet"
  }
}
```

### Example 2: All Local with MLX

```json5
{
  "Providers": [
    {
      "name": "mlx",
      "api_base_url": "http://127.0.0.1:3457/v1/chat/completions",
      "api_key": "mlx",
      "models": [
        "qwen2.5-coder-7b",
        "llama-3.1-8b",
        "deepseek-r1-7b"
      ],
      "transformer": {"use": ["mlx"]}
    }
  ],
  "Router": {
    "default": "mlx,qwen2.5-coder-7b",
    "background": "mlx,qwen2.5-coder-7b",
    "think": "mlx,deepseek-r1-7b",
    "longContext": "mlx,llama-3.1-8b"
  },
  "MLX": {
    "models_dir": "~/.claude-code-router/models"
  }
}
```

## Performance Optimization

### Model Quantization

Use 4-bit quantized models for best performance/quality trade-off:
- 4-bit: Best performance, good quality
- 8-bit: Better quality, slower
- FP16: Best quality, slowest

### Memory Management

```python
# In MLX server, implement model unloading
def unload_model(self, model_name: str):
    """Unload a model to free memory."""
    if model_name in self.loaded_models:
        del self.loaded_models[model_name]
        mx.metal.clear_cache()  # Clear MLX metal cache
```

### Caching

Cache model outputs for identical requests:

```python
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_generate(prompt_hash, model, max_tokens, temperature):
    # ... generation logic
    pass
```

## Benefits of Python + MLX Implementation

1. **Native MLX support**: Python is the primary language for MLX
2. **Direct model loading**: No need for separate services
3. **Unified memory**: MLX leverages Apple Silicon's unified memory
4. **Cost savings**: No API costs for local inference
5. **Privacy**: All inference happens locally
6. **Flexibility**: Can mix local and cloud models easily

## Recommended Workflow

1. Use **local MLX models** for:
   - Background tasks (file searching, grepping)
   - Simple code generation
   - Quick edits
   - Testing/experimentation

2. Use **cloud models** for:
   - Complex reasoning (Plan Mode)
   - Long context analysis
   - Critical production code
   - Tasks requiring latest models

This hybrid approach gives you the best of both worlds: cost-effective local inference for routine tasks, and powerful cloud models when you need them.
