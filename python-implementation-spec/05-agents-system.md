# Agents System Specification

## Overview

The agent system extends Claude Code Router's capabilities by intercepting tool calls and providing custom implementations. The primary use case is the Image Agent, which allows non-vision models to analyze images by routing image analysis to a vision-capable model.

## Agent Architecture

### Agent Interface

```python
from abc import ABC, abstractmethod
from typing import Dict, Optional

class Tool:
    """Represents a tool that can be called by the LLM."""
    def __init__(self, name: str, description: str, input_schema: dict, handler):
        self.name = name
        self.description = description
        self.input_schema = input_schema
        self.handler = handler  # async function(args, context) -> str

class Agent(ABC):
    """Base agent interface."""

    def __init__(self):
        self.name: str = ""
        self.tools: Dict[str, Tool] = {}

    @abstractmethod
    def should_handle(self, request: Request, config: Config) -> bool:
        """Determine if this agent should handle the request."""
        pass

    @abstractmethod
    def req_handler(self, request: Request, config: Config) -> None:
        """Modify the request before it's sent to the LLM."""
        pass

    def res_handler(self, response: dict, config: Config) -> Optional[dict]:
        """Optionally modify the response (rarely used)."""
        return None
```

## Image Agent Implementation

### Purpose

Enable non-vision models to "see" images by:
1. Detecting images in the conversation
2. Caching image data
3. Replacing images with placeholders
4. Injecting an `analyzeImage` tool
5. When tool is called, routing to a vision model
6. Returning analysis results

### Image Agent Code

```python
import hashlib
from lru_cache import LRUCache

class ImageCache:
    """Cache for storing image data temporarily."""

    def __init__(self, max_size: int = 100):
        self.cache = LRUCache(
            max_size=max_size,
            ttl=5 * 60  # 5 minutes
        )

    def store_image(self, id: str, source: dict) -> None:
        if self.has_image(id):
            return
        self.cache.set(id, {
            "source": source,
            "timestamp": time.time()
        })

    def get_image(self, id: str) -> Optional[dict]:
        entry = self.cache.get(id)
        return entry["source"] if entry else None

    def has_image(self, id: str) -> bool:
        return self.cache.has(id)

image_cache = ImageCache()


class ImageAgent(Agent):
    """Agent for handling image analysis."""

    def __init__(self):
        super().__init__()
        self.name = "image"
        self._setup_tools()

    def _setup_tools(self):
        """Setup the analyzeImage tool."""
        self.tools["analyzeImage"] = Tool(
            name="analyzeImage",
            description="Analyze image or images by ID and extract information such as OCR text, objects, layout, colors, or safety signals.",
            input_schema={
                "type": "object",
                "properties": {
                    "imageId": {
                        "type": "array",
                        "description": "an array of IDs to analyze",
                        "items": {"type": "string"}
                    },
                    "task": {
                        "type": "string",
                        "description": "Details of task to perform on the image. The more detailed, the better"
                    },
                    "regions": {
                        "type": "array",
                        "description": "Optional regions of interest within the image",
                        "items": {
                            "type": "object",
                            "properties": {
                                "name": {"type": "string"},
                                "x": {"type": "number"},
                                "y": {"type": "number"},
                                "w": {"type": "number"},
                                "h": {"type": "number"},
                                "units": {"type": "string", "enum": ["px", "pct"]}
                            },
                            "required": ["x", "y", "w", "h", "units"]
                        }
                    }
                },
                "required": ["imageId", "task"]
            },
            handler=self.analyze_image
        )

    def should_handle(self, request: Request, config: Config) -> bool:
        """
        Determine if this request contains images that need processing.

        Rules:
        1. If Router.image is not configured, don't handle
        2. If model is already the image model, don't handle
        3. If forceUseImageAgent is true, always handle if images present
        4. Otherwise, handle if images are NOT in the last message
           (meaning images are in history, not current input)
        """
        if not config.Router.image:
            return False

        if request.body.get("model") == config.Router.image:
            return False

        # Check for images in messages
        messages = request.body.get("messages", [])
        has_images = any(
            self._message_has_images(msg)
            for msg in messages
        )

        if not has_images:
            return False

        # If last message has images, optionally route to vision model directly
        last_message = messages[-1] if messages else None
        if (not config.forceUseImageAgent and
            last_message and
            last_message.get("role") == "user" and
            self._message_has_images(last_message)):
            # Route to vision model directly (no agent)
            request.body["model"] = config.Router.image
            # Clean up tool_result images to avoid bloat
            self._clean_tool_result_images(last_message)
            return False

        # Images are in history, use agent
        return True

    def _message_has_images(self, message: dict) -> bool:
        """Check if a message contains images."""
        content = message.get("content", [])
        if isinstance(content, str):
            return False

        for item in content:
            if item.get("type") == "image":
                return True
            if isinstance(item.get("content"), list):
                if any(sub.get("type") == "image" for sub in item["content"]):
                    return True
        return False

    def _clean_tool_result_images(self, message: dict):
        """Replace tool_result images with placeholder text."""
        content = message.get("content", [])
        if isinstance(content, list):
            for item in content:
                if item.get("type") == "tool_result" and isinstance(item.get("content"), list):
                    if any(e.get("type") == "image" for e in item["content"]):
                        item["content"] = "read image successfully"

    def req_handler(self, request: Request, config: Config) -> None:
        """
        Modify the request to replace images with placeholders.

        Process:
        1. Find all messages with images
        2. For each image:
           a. Generate a unique ID
           b. Cache the image data
           c. Replace image with text placeholder "[Image #ID]"
        3. Inject system prompt explaining image handling
        """
        # Inject system prompt
        system = request.body.get("system", [])
        if not isinstance(system, list):
            system = [{"type": "text", "text": system}] if system else []

        system.append({
            "type": "text",
            "text": """You are a text-only language model and do not possess visual perception.
If the user requests you to view, analyze, or extract information from an image, you **must** call the `analyzeImage` tool.

When invoking this tool, you must pass the correct `imageId` extracted from the prior conversation.
Image identifiers are always provided in the format `[Image #imageId]`.

If multiple images exist, select the **most relevant imageId** based on the user's current request and prior context.

Do not attempt to describe or analyze the image directly yourself.
Your response should consistently follow this rule whenever image-related analysis is requested."""
        })
        request.body["system"] = system

        # Find and process images
        messages = request.body.get("messages", [])
        image_id = 1

        for message in messages:
            if message.get("role") != "user":
                continue

            content = message.get("content", [])
            if not isinstance(content, list):
                continue

            for item in content:
                if item.get("type") == "image":
                    # Cache image
                    cache_key = f"{request.id}_Image#{image_id}"
                    image_cache.store_image(cache_key, item["source"])

                    # Replace with placeholder
                    item["type"] = "text"
                    del item["source"]
                    item["text"] = f"[Image #{image_id}]This is an image, if you need to view or analyze it, you need to extract the imageId"

                    image_id += 1

                elif item.get("type") == "tool_result":
                    # Handle images in tool results
                    tool_content = item.get("content", [])
                    if isinstance(tool_content, list):
                        for sub_item in tool_content:
                            if sub_item.get("type") == "image":
                                cache_key = f"{request.id}_Image#{image_id}"
                                image_cache.store_image(cache_key, sub_item["source"])

                                item["content"] = f"[Image #{image_id}]This is an image, if you need to view or analyze it, you need to extract the imageId"
                                image_id += 1
                                break

    async def analyze_image(self, args: dict, context: dict) -> str:
        """
        Handle the analyzeImage tool call.

        Args:
            args: Tool arguments (imageId, task, regions)
            context: {req: Request, config: Config}

        Returns:
            Image analysis result as text
        """
        req = context["req"]
        config = context["config"]

        # Retrieve cached images
        image_messages = []
        image_ids = args.get("imageId", [])
        if not isinstance(image_ids, list):
            image_ids = [image_ids]

        for img_id in image_ids:
            cache_key = f"{req.id}_Image#{img_id}"
            image_source = image_cache.get_image(cache_key)
            if image_source:
                image_messages.append({
                    "type": "image",
                    "source": image_source
                })

        # Add task description
        if args.get("task"):
            image_messages.append({
                "type": "text",
                "text": args["task"]
            })

        # Add any additional args as context
        extra_args = {k: v for k, v in args.items() if k not in ["imageId", "task"]}
        if extra_args:
            image_messages.append({
                "type": "text",
                "text": json.dumps(extra_args)
            })

        # Call vision model
        port = config.PORT or 3456
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"http://127.0.0.1:{port}/v1/messages",
                headers={
                    "x-api-key": config.APIKEY or "",
                    "content-type": "application/json"
                },
                json={
                    "model": config.Router.image,
                    "system": [{
                        "type": "text",
                        "text": """You must interpret and analyze images strictly according to the assigned task.
When an image placeholder is provided, your role is to parse the image content only within the scope of the user's instructions.
Do not ignore or deviate from the task.
Always ensure that your response reflects a clear, accurate interpretation of the image aligned with the given objective."""
                    }],
                    "messages": [{
                        "role": "user",
                        "content": image_messages
                    }],
                    "stream": False
                }
            )

            if response.status_code != 200:
                return "analyzeImage Error"

            result = response.json()
            if not result.get("content"):
                return "analyzeImage Error"

            return result["content"][0]["text"]
```

## Agent Manager

```python
class AgentsManager:
    """Manages all registered agents."""

    def __init__(self):
        self.agents: Dict[str, Agent] = {}

    def register_agent(self, agent: Agent) -> None:
        """Register an agent."""
        self.agents[agent.name] = agent

    def get_agent(self, name: str) -> Optional[Agent]:
        """Get an agent by name."""
        return self.agents.get(name)

    def get_all_agents(self) -> List[Agent]:
        """Get all registered agents."""
        return list(self.agents.values())

    def get_all_tools(self) -> List[Tool]:
        """Get all tools from all agents."""
        tools = []
        for agent in self.agents.values():
            tools.extend(agent.tools.values())
        return tools


# Global agents manager
agents_manager = AgentsManager()
agents_manager.register_agent(ImageAgent())
```

## Agent Stream Processing

When an agent is active, tool calls in the response stream must be intercepted:

```python
async def process_agent_stream(request: Request, response_stream):
    """
    Process streaming response with agent tool interception.

    Flow:
    1. Parse SSE stream
    2. Detect tool_use events
    3. If tool belongs to an agent, execute it
    4. Continue streaming or make recursive call
    """
    from .sse import SSEParser, SSESerializer

    # Parse SSE to objects
    parser = SSEParser()
    events = parser.parse_stream(response_stream)

    # Track tool call state
    current_tool = None
    current_tool_args = ""
    assistant_messages = []
    tool_results = []

    async for event in events:
        # Detect tool call start
        if (event.get("event") == "content_block_start" and
            event.get("data", {}).get("content_block", {}).get("name")):

            tool_name = event["data"]["content_block"]["name"]
            tool_id = event["data"]["content_block"]["id"]

            # Check if tool belongs to an agent
            agent = None
            for agent_name in request.state.agents:
                a = agents_manager.get_agent(agent_name)
                if a and tool_name in a.tools:
                    agent = a
                    break

            if agent:
                current_tool = {
                    "name": tool_name,
                    "id": tool_id,
                    "agent": agent,
                    "index": event["data"]["index"]
                }
                continue  # Don't yield this event

        # Collect tool arguments
        if current_tool and event.get("data", {}).get("delta", {}).get("type") == "input_json_delta":
            current_tool_args += event["data"]["delta"]["partial_json"]
            continue

        # Tool call complete
        if current_tool and event.get("event") == "content_block_stop":
            # Parse args
            args = json.loads(current_tool_args)

            # Execute agent tool
            result = await current_tool["agent"].tools[current_tool["name"]].handler(
                args,
                {"req": request, "config": request.app.state.config}
            )

            # Store for next call
            assistant_messages.append({
                "type": "tool_use",
                "id": current_tool["id"],
                "name": current_tool["name"],
                "input": args
            })
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": current_tool["id"],
                "content": result
            })

            current_tool = None
            current_tool_args = ""
            continue

        # If we have tool results and message is ending, make recursive call
        if tool_results and event.get("event") == "message_delta":
            # Make another API call with tool results
            request.body["messages"].append({
                "role": "assistant",
                "content": assistant_messages
            })
            request.body["messages"].append({
                "role": "user",
                "content": tool_results
            })

            # Recursive call
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    f"http://127.0.0.1:{config.PORT}/v1/messages",
                    headers={
                        "x-api-key": config.APIKEY,
                        "content-type": "application/json"
                    },
                    json=request.body,
                    stream=True
                )

                # Stream the recursive call's response
                async for line in response.aiter_lines():
                    if line.startswith("data: "):
                        data = json.loads(line[6:])
                        # Skip message_start and message_stop
                        if data.get("type") in ["message_start", "message_stop"]:
                            continue
                        yield line + "\n\n"

            return

        # Normal event, yield it
        yield SSESerializer.serialize(event)
```

## Python Implementation Notes

The agent system requires:
- LRU cache for image storage
- Ability to intercept and modify streams
- Recursive API calls
- JSON5 parsing for tool arguments
