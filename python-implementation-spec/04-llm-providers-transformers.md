# LLM Providers and Transformers Specification

## Overview

The provider system handles communication with different LLM APIs and transforms requests/responses to match each provider's expected format. This is the core abstraction that allows Claude Code Router to work with many different LLM services.

## Provider Abstraction

### Provider Definition

```python
@dataclass
class Provider:
    name: str                    # Unique identifier (e.g., "openrouter", "deepseek")
    api_base_url: str           # API endpoint
    api_key: str                # Authentication key
    models: List[str]           # Available model names
    transformer: Optional[TransformerConfig] = None  # Request/response transformers
```

### Making API Calls

```python
async def call_provider_api(
    provider: Provider,
    model: str,
    request_body: dict,
    stream: bool = False,
    timeout: int = 600000
) -> httpx.Response:
    """
    Make an HTTP request to a provider's API.

    Args:
        provider: Provider configuration
        model: Model name
        request_body: Transformed request body
        stream: Whether to stream the response
        timeout: Request timeout in milliseconds

    Returns:
        HTTP response (streaming or complete)
    """
    import httpx

    headers = {
        "Authorization": f"Bearer {provider.api_key}",
        "Content-Type": "application/json"
    }

    # Handle proxy if configured
    proxies = None
    if config.PROXY_URL:
        proxies = {
            "http://": config.PROXY_URL,
            "https://": config.PROXY_URL
        }

    async with httpx.AsyncClient(
        proxies=proxies,
        timeout=timeout / 1000  # Convert to seconds
    ) as client:
        if stream:
            # Streaming request
            response = await client.post(
                provider.api_base_url,
                json=request_body,
                headers=headers,
                stream=True
            )
            return response
        else:
            # Normal request
            response = await client.post(
                provider.api_base_url,
                json=request_body,
                headers=headers
            )
            return response
```

## Transformer System

### Purpose

Different LLM providers have different API formats. Transformers adapt:

1. **Request transformation**: Convert Anthropic format → Provider format
2. **Response transformation**: Convert Provider format → Anthropic format

### Transformer Interface

```python
from abc import ABC, abstractmethod

class Transformer(ABC):
    """Base transformer interface."""

    def __init__(self, options: dict = None):
        self.options = options or {}

    @abstractmethod
    async def transform_request(self, request: dict) -> dict:
        """Transform request from Anthropic format to provider format."""
        pass

    @abstractmethod
    async def transform_response(self, response: dict) -> dict:
        """Transform response from provider format to Anthropic format."""
        pass

    async def transform_stream(self, stream: AsyncIterator) -> AsyncIterator:
        """Transform streaming response. Default: no transformation."""
        async for chunk in stream:
            yield chunk
```

### Built-in Transformers

#### 1. Anthropic Transformer (Pass-through)

```python
class AnthropicTransformer(Transformer):
    """
    Pass-through transformer for Anthropic API.
    Used when connecting directly to Anthropic.
    """

    async def transform_request(self, request: dict) -> dict:
        return request  # No transformation needed

    async def transform_response(self, response: dict) -> dict:
        return response
```

#### 2. DeepSeek Transformer

```python
class DeepSeekTransformer(Transformer):
    """
    Transformer for DeepSeek API.

    Request changes:
    - Anthropic format → OpenAI format
    - Handle system prompt differences
    - Remove unsupported fields

    Response changes:
    - OpenAI format → Anthropic format
    - Map choice[0].message → content
    """

    async def transform_request(self, request: dict) -> dict:
        transformed = {
            "model": request["model"],
            "max_tokens": request.get("max_tokens", 1024),
            "temperature": request.get("temperature", 1.0),
            "stream": request.get("stream", False),
            "messages": []
        }

        # Convert system prompt to message
        if "system" in request:
            system_text = self._extract_system_text(request["system"])
            if system_text:
                transformed["messages"].append({
                    "role": "system",
                    "content": system_text
                })

        # Convert messages
        for msg in request.get("messages", []):
            transformed["messages"].append({
                "role": msg["role"],
                "content": self._convert_content(msg["content"])
            })

        # Add tools if present
        if "tools" in request:
            transformed["tools"] = self._convert_tools(request["tools"])

        return transformed

    def _extract_system_text(self, system) -> str:
        if isinstance(system, str):
            return system
        if isinstance(system, list):
            return "\n".join(item.get("text", "") for item in system if item.get("type") == "text")
        return ""

    def _convert_content(self, content):
        if isinstance(content, str):
            return content
        # Handle complex content blocks
        # ... conversion logic
        return content

    async def transform_response(self, response: dict) -> dict:
        # Convert OpenAI format to Anthropic format
        return {
            "id": response.get("id"),
            "type": "message",
            "role": "assistant",
            "content": [{
                "type": "text",
                "text": response["choices"][0]["message"]["content"]
            }],
            "model": response.get("model"),
            "usage": {
                "input_tokens": response["usage"]["prompt_tokens"],
                "output_tokens": response["usage"]["completion_tokens"]
            }
        }
```

#### 3. Gemini Transformer

```python
class GeminiTransformer(Transformer):
    """
    Transformer for Google Gemini API.

    Gemini has a unique format:
    - Different endpoint structure (model in URL)
    - Different message format
    - Different tool format
    """

    async def transform_request(self, request: dict) -> dict:
        model = request.get("model", "gemini-pro")

        transformed = {
            "contents": self._convert_messages(request.get("messages", [])),
            "generationConfig": {
                "maxOutputTokens": request.get("max_tokens", 1024),
                "temperature": request.get("temperature", 1.0),
            }
        }

        # Add system instruction
        if "system" in request:
            transformed["systemInstruction"] = {
                "parts": [{"text": self._extract_system_text(request["system"])}]
            }

        # Add tools
        if "tools" in request:
            transformed["tools"] = self._convert_tools(request["tools"])

        return transformed

    def _convert_messages(self, messages: list) -> list:
        contents = []
        for msg in messages:
            contents.append({
                "role": "user" if msg["role"] == "user" else "model",
                "parts": self._convert_content_parts(msg["content"])
            })
        return contents

    async def transform_response(self, response: dict) -> dict:
        # Convert Gemini format to Anthropic format
        candidate = response["candidates"][0]
        content = candidate["content"]

        return {
            "id": "msg_" + uuid.uuid4().hex[:8],
            "type": "message",
            "role": "assistant",
            "content": [{
                "type": "text",
                "text": content["parts"][0]["text"]
            }],
            "usage": {
                "input_tokens": response["usageMetadata"]["promptTokenCount"],
                "output_tokens": response["usageMetadata"]["candidatesTokenCount"]
            }
        }
```

#### 4. OpenRouter Transformer

```python
class OpenRouterTransformer(Transformer):
    """
    Transformer for OpenRouter API.

    OpenRouter uses OpenAI-compatible format with extensions.
    """

    def __init__(self, options: dict = None):
        super().__init__(options)
        # Provider routing options
        self.provider_routing = options.get("provider", {}) if options else {}

    async def transform_request(self, request: dict) -> dict:
        transformed = {
            "model": request["model"],
            "messages": request.get("messages", []),
            "max_tokens": request.get("max_tokens"),
            "temperature": request.get("temperature"),
            "stream": request.get("stream", False)
        }

        # Add OpenRouter-specific provider routing
        if self.provider_routing:
            transformed["provider"] = self.provider_routing

        # Add system prompt as first message
        if "system" in request:
            system_text = self._extract_system_text(request["system"])
            if system_text:
                transformed["messages"].insert(0, {
                    "role": "system",
                    "content": system_text
                })

        return transformed
```

#### 5. MaxToken Transformer

```python
class MaxTokenTransformer(Transformer):
    """
    Transformer that sets a specific max_tokens value.
    """

    def __init__(self, options: dict):
        super().__init__(options)
        self.max_tokens = options.get("max_tokens", 30000)

    async def transform_request(self, request: dict) -> dict:
        request["max_tokens"] = self.max_tokens
        return request

    async def transform_response(self, response: dict) -> dict:
        return response  # No transformation
```

#### 6. ToolUse Transformer

```python
class ToolUseTransformer(Transformer):
    """
    Transformer that optimizes tool usage.

    Sets tool_choice to force the model to use tools when available.
    """

    async def transform_request(self, request: dict) -> dict:
        if "tools" in request and request["tools"]:
            # Force tool use for better reliability
            request["tool_choice"] = {"type": "auto"}
        return request

    async def transform_response(self, response: dict) -> dict:
        return response
```

#### 7. Reasoning Transformer

```python
class ReasoningTransformer(Transformer):
    """
    Transformer for models that output reasoning_content.

    Some models (like DeepSeek-R1) output thinking process in a
    separate field that should be included in the response.
    """

    async def transform_response(self, response: dict) -> dict:
        # Check if response has reasoning_content
        if "reasoning_content" in response:
            # Add reasoning as a separate content block
            reasoning_text = response["reasoning_content"]
            content = response.get("content", [])

            # Prepend reasoning
            content.insert(0, {
                "type": "text",
                "text": f"[Reasoning]\n{reasoning_text}\n\n"
            })

            response["content"] = content

        return response
```

### Transformer Chaining

Transformers can be chained together. They're applied in order:

```python
class TransformerChain:
    """Chain multiple transformers together."""

    def __init__(self, transformers: List[Transformer]):
        self.transformers = transformers

    async def transform_request(self, request: dict) -> dict:
        for transformer in self.transformers:
            request = await transformer.transform_request(request)
        return request

    async def transform_response(self, response: dict) -> dict:
        # Apply transformers in reverse order for responses
        for transformer in reversed(self.transformers):
            response = await transformer.transform_response(response)
        return response

    async def transform_stream(self, stream: AsyncIterator) -> AsyncIterator:
        for transformer in reversed(self.transformers):
            stream = transformer.transform_stream(stream)
        return stream
```

### Loading Transformers

```python
def load_transformers(provider: Provider, model: str) -> TransformerChain:
    """
    Load transformers for a provider/model combination.

    Transformer precedence:
    1. Model-specific transformers
    2. Global provider transformers

    Example config:
    {
      "transformer": {
        "use": ["deepseek", ["maxtoken", {"max_tokens": 30000}]],
        "deepseek-chat": {
          "use": ["tooluse"]
        }
      }
    }
    """
    transformers = []

    if not provider.transformer:
        return TransformerChain([])

    # Load global transformers
    global_transformers = provider.transformer.get("use", [])
    for t in global_transformers:
        transformers.append(create_transformer(t))

    # Load model-specific transformers
    if model in provider.transformer:
        model_transformers = provider.transformer[model].get("use", [])
        for t in model_transformers:
            transformers.append(create_transformer(t))

    return TransformerChain(transformers)


def create_transformer(spec: Union[str, Tuple[str, dict]]) -> Transformer:
    """
    Create a transformer from a specification.

    Args:
        spec: Either a string name or tuple of (name, options)

    Example:
        "deepseek" → DeepSeekTransformer()
        ["maxtoken", {"max_tokens": 30000}] → MaxTokenTransformer(...)
    """
    if isinstance(spec, str):
        name = spec
        options = {}
    else:
        name, options = spec

    transformer_map = {
        "anthropic": AnthropicTransformer,
        "deepseek": DeepSeekTransformer,
        "gemini": GeminiTransformer,
        "openrouter": OpenRouterTransformer,
        "groq": GroqTransformer,
        "maxtoken": MaxTokenTransformer,
        "tooluse": ToolUseTransformer,
        "reasoning": ReasoningTransformer,
        # ... more transformers
    }

    transformer_class = transformer_map.get(name)
    if not transformer_class:
        raise ValueError(f"Unknown transformer: {name}")

    return transformer_class(options)
```

## Custom Transformers

Users can create custom transformers and load them from plugins:

```python
# ~/.claude-code-router/plugins/my_transformer.py
from claude_code_router.transformers import Transformer

class MyCustomTransformer(Transformer):
    def __init__(self, options: dict = None):
        super().__init__(options)
        self.my_option = options.get("my_option", "default") if options else "default"

    async def transform_request(self, request: dict) -> dict:
        # Custom transformation logic
        request["custom_field"] = self.my_option
        return request

    async def transform_response(self, response: dict) -> dict:
        # Custom transformation logic
        return response

# Export the transformer
transformer = MyCustomTransformer
```

Configuration:
```json5
{
  "transformers": [
    {
      "path": "~/.claude-code-router/plugins/my_transformer.py",
      "options": {
        "my_option": "custom_value"
      }
    }
  ]
}
```

## Python Implementation Recommendations

### Libraries

- **httpx**: Async HTTP client with streaming support
  ```bash
  pip install httpx
  ```

- **abc**: Abstract base classes (standard library)

### Structure

```
providers/
├── __init__.py
├── base.py           # Base Provider and Transformer classes
├── client.py         # HTTP client for API calls
└── transformers/
    ├── __init__.py
    ├── anthropic.py
    ├── deepseek.py
    ├── gemini.py
    ├── openrouter.py
    └── ...
```

### Example Usage

```python
# Get provider config
provider = config.get_provider("deepseek")

# Load transformers
transformers = load_transformers(provider, "deepseek-chat")

# Transform request
original_request = {...}  # Anthropic format
transformed_request = await transformers.transform_request(original_request)

# Call API
response = await call_provider_api(provider, "deepseek-chat", transformed_request)

# Transform response
response_data = await response.json()
anthropic_response = await transformers.transform_response(response_data)
```
