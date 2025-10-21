# Streaming and Server-Sent Events (SSE)

## Overview

Claude Code uses Server-Sent Events (SSE) for streaming responses. The router must:
1. Parse incoming SSE streams from providers
2. Transform event data
3. Re-serialize to SSE format
4. Stream to client

## SSE Format

### Structure

```
event: <event_type>
data: <json_data>

```

**Note**: Each event ends with a blank line (`\n\n`)

### Example SSE Stream

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01ABC","type":"message","role":"assistant","content":[],"model":"claude-3-5-sonnet","usage":{"input_tokens":100,"output_tokens":0}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" world"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":2}}

event: message_stop
data: {"type":"message_stop"}

```

## SSE Parser (Transform)

### Purpose

Convert SSE text stream → Python objects

### Implementation

```python
class SSEParser:
    """Parse Server-Sent Events stream."""

    def __init__(self):
        self.buffer = ""
        self.current_event = {}

    async def parse_stream(self, stream: AsyncIterator[bytes]) -> AsyncIterator[dict]:
        """
        Parse an SSE stream into event objects.

        Yields:
            Event dictionaries with 'event' and 'data' keys
        """
        async for chunk in stream:
            # Decode chunk
            text = chunk.decode('utf-8')
            self.buffer += text

            # Split by lines
            lines = self.buffer.split('\n')
            self.buffer = lines.pop()  # Keep incomplete line

            for line in lines:
                event = self._process_line(line)
                if event:
                    yield event

        # Handle remaining buffer
        if self.buffer.strip():
            event = self._process_line(self.buffer.strip())
            if event:
                yield event

        # Yield final event if any
        if self.current_event:
            yield self.current_event

    def _process_line(self, line: str) -> Optional[dict]:
        """Process a single line."""
        line = line.strip()

        if not line:
            # Empty line = event complete
            if self.current_event:
                event = self.current_event.copy()
                self.current_event = {}
                return event
            return None

        if line.startswith('event:'):
            self.current_event['event'] = line[6:].strip()

        elif line.startswith('data:'):
            data_str = line[5:].strip()
            if data_str == '[DONE]':
                self.current_event['data'] = {'type': 'done'}
            else:
                try:
                    self.current_event['data'] = json.loads(data_str)
                except json.JSONDecodeError:
                    self.current_event['data'] = {'raw': data_str, 'error': 'JSON parse failed'}

        elif line.startswith('id:'):
            self.current_event['id'] = line[3:].strip()

        elif line.startswith('retry:'):
            self.current_event['retry'] = int(line[6:].strip())

        return None
```

## SSE Serializer (Transform)

### Purpose

Convert Python objects → SSE text stream

### Implementation

```python
class SSESerializer:
    """Serialize events to Server-Sent Events format."""

    @staticmethod
    def serialize(event: dict) -> str:
        """
        Serialize an event object to SSE format.

        Args:
            event: Dictionary with 'event', 'data', 'id', 'retry' keys

        Returns:
            SSE-formatted string
        """
        output = ""

        if 'event' in event:
            output += f"event: {event['event']}\n"

        if 'id' in event:
            output += f"id: {event['id']}\n"

        if 'retry' in event:
            output += f"retry: {event['retry']}\n"

        if 'data' in event:
            if event['data'].get('type') == 'done':
                output += "data: [DONE]\n"
            else:
                output += f"data: {json.dumps(event['data'])}\n"

        output += "\n"
        return output

    @staticmethod
    async def serialize_stream(events: AsyncIterator[dict]) -> AsyncIterator[str]:
        """Serialize a stream of events."""
        async for event in events:
            yield SSESerializer.serialize(event)
```

## Stream Rewriting

### Purpose

Process a stream, applying transformations to each event

### Implementation

```python
async def rewrite_stream(
    stream: AsyncIterator,
    processor: Callable[[dict, Any], Awaitable[Optional[dict]]]
) -> AsyncIterator:
    """
    Read source stream, process each event, yield transformed events.

    Args:
        stream: Source event stream
        processor: async function(event, controller) -> Optional[event]
                   If returns None, event is not yielded

    Yields:
        Processed events
    """
    async for event in stream:
        processed = await processor(event, None)
        if processed is not None:
            yield processed
```

## FastAPI Streaming Response

### Example

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

@app.post("/v1/messages")
async def handle_messages(request: Request):
    body = await request.json()

    # ... routing logic ...

    if body.get("stream"):
        async def generate():
            # Get provider response stream
            response = await call_provider_api(...)

            # Parse SSE
            parser = SSEParser()
            events = parser.parse_stream(response.aiter_bytes())

            # Apply transformers
            transformed = transform_stream(events, transformers)

            # Serialize back to SSE
            async for sse_text in SSESerializer.serialize_stream(transformed):
                yield sse_text

        return StreamingResponse(
            generate(),
            media_type="text/event-stream"
        )
```

## Stream Tee-ing for Usage Tracking

### Purpose

Read a stream twice: once for response, once for usage tracking

### Implementation

```python
import asyncio
from typing import AsyncIterator

async def tee_stream(stream: AsyncIterator) -> tuple[AsyncIterator, AsyncIterator]:
    """
    Split a stream into two independent streams.

    Note: This is complex in async Python. Alternative approach:
    - Parse stream once
    - Yield to client
    - Track usage in same pass
    """
    # Simpler approach: track while yielding
    async def track_and_yield(stream):
        async for event in stream:
            # Track usage if message_delta event
            if event.get('event') == 'message_delta':
                usage = event.get('data', {}).get('usage')
                if usage:
                    session_cache.put(session_id, Usage(**usage))

            yield event

    return track_and_yield(stream)
```

## Event Types

### Anthropic SSE Events

1. **message_start**: Start of message
2. **content_block_start**: Start of content block (text or tool_use)
3. **content_block_delta**: Delta update to content block
4. **content_block_stop**: End of content block
5. **message_delta**: Message metadata update (usage, stop_reason)
6. **message_stop**: End of message
7. **error**: Error occurred

### Tool Use Events

```json
{
  "event": "content_block_start",
  "data": {
    "type": "content_block_start",
    "index": 0,
    "content_block": {
      "type": "tool_use",
      "id": "tool_123",
      "name": "get_weather",
      "input": {}
    }
  }
}

{
  "event": "content_block_delta",
  "data": {
    "type": "content_block_delta",
    "index": 0,
    "delta": {
      "type": "input_json_delta",
      "partial_json": "{\"location\""
    }
  }
}

{
  "event": "content_block_delta",
  "data": {
    "type": "content_block_delta",
    "index": 0,
    "delta": {
      "type": "input_json_delta",
      "partial_json": ": \"San Francisco\"}"
    }
  }
}

{
  "event": "content_block_stop",
  "data": {
    "type": "content_block_stop",
    "index": 0
  }
}
```

## Python Recommendations

### Libraries

- **FastAPI StreamingResponse**: Built-in streaming support
- **httpx**: Async HTTP client with streaming
  ```python
  async with httpx.AsyncClient() as client:
      async with client.stream("POST", url, json=data) as response:
          async for line in response.aiter_lines():
              # Process SSE line
  ```

### Testing Streaming

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_streaming():
    async with AsyncClient(app=app, base_url="http://test") as client:
        async with client.stream(
            "POST",
            "/v1/messages",
            json={"model": "test,model", "stream": True, "messages": [...]}
        ) as response:
            events = []
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    events.append(json.loads(line[6:]))

            assert len(events) > 0
            assert events[0]["type"] == "message_start"
```
