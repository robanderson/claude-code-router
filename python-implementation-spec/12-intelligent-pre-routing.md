# Intelligent Pre-Routing with Small Fast Models

## Overview

This document specifies an advanced enhancement to the routing system: using a small, ultra-fast model (1B-3B parameters) to pre-classify requests and make intelligent routing decisions before engaging larger models.

## Motivation

### The Problem

In the basic routing system, all requests go through the same decision tree based on heuristics (token count, model type, etc.). This means:

- **Expensive models handle simple tasks**: Sonnet 4.5 lists files, formats code, creates git commits
- **Wasted tokens**: ~60-80% of requests could be handled by cheaper/local models
- **Slower responses**: Cloud API latency for tasks that could run instantly locally
- **Higher costs**: $0.003/1K tokens for work a local model could do free

### The Solution

Use a tiny, fast classification model to analyze each request and route intelligently:

```
Request → Pre-Router (1.5B local, 50-100ms) → Route Decision → Appropriate Model
```

**Benefits**:
- 89% cost reduction (estimated)
- 40% faster average response time
- Better resource utilization
- Seamless user experience

## Pre-Router Model Characteristics

### Model Selection

**Recommended models**:
- **Qwen2.5-1.5B-Instruct** - Excellent reasoning, 200-400 tok/s
- **Llama-3.2-3B-Instruct** - Strong classification, 150-300 tok/s
- **Phi-3-mini** (3.8B) - Good balance, 100-200 tok/s

**Requirements**:
- Fast inference (>150 tok/s on M2 Ultra)
- Good instruction following
- Small memory footprint (<2GB)
- Strong text classification ability

### Performance Targets

On M2 Ultra with 4-bit quantization:
- **Latency**: 50-100ms for classification
- **Throughput**: 200-400 tokens/sec
- **Memory**: 1-2GB VRAM
- **Accuracy**: >95% correct routing decisions

## Classification Categories

### Route Types

```python
from enum import Enum

class RouteDecision(Enum):
    """Possible routing decisions."""

    # Tool-only execution (no LLM)
    TOOL_ONLY = "tool_only"           # grep, ls, git status

    # Local models (MLX)
    LOCAL_TINY = "local_tiny"         # 1-2B: Syntax fixes, simple renames
    LOCAL_FAST = "local_fast"         # 7B: File ops, git commits, formatting
    LOCAL_MEDIUM = "local_medium"     # 14B: Simple refactoring, tests
    LOCAL_LARGE = "local_large"       # 30B: Complex code generation

    # Cloud models (cheap)
    CLOUD_HAIKU = "cloud_haiku"       # Simple questions, summaries

    # Cloud models (expensive - use sparingly)
    CLOUD_SONNET = "cloud_sonnet"     # Complex reasoning, architecture
    CLOUD_OPUS = "cloud_opus"         # Critical production code

    # Special routing
    BACKGROUND = "background"          # Low-priority background work
    THINKING = "thinking"              # Plan mode, deep reasoning
```

## High-Value Routing Use Cases

### 1. File System Operations

**Query patterns**:
- "List all Python files in src/"
- "Count lines of code in this project"
- "Find files containing 'TODO'"
- "Show directory structure"

**Classification**:
```python
{
    "route": "LOCAL_FAST" or "TOOL_ONLY",
    "confidence": 0.95,
    "reasoning": "Pure file system operation, no creativity needed"
}
```

**Token savings**: ~1000 tokens per request
**Speed improvement**: 10x faster (instant vs 2-3 second API call)

### 2. Git Operations

**Query patterns**:
- "Create a commit with a descriptive message"
- "Show git status"
- "List recent commits"
- "Create a branch"

**Classification**:
```python
{
    "route": "LOCAL_FAST",
    "confidence": 0.90,
    "reasoning": "Git housekeeping, conventional patterns"
}
```

**Why not Sonnet**: Conventional commits follow patterns, don't need SOTA reasoning

### 3. Code Formatting/Linting

**Query patterns**:
- "Format this file according to PEP 8"
- "Fix linting errors"
- "Organize imports"

**Classification**:
```python
{
    "route": "TOOL_ONLY",
    "confidence": 0.99,
    "reasoning": "Deterministic tools (black, ruff, isort) handle this"
}
```

**Token savings**: 100% - bypass LLM entirely

### 4. Simple Refactoring

**Query patterns**:
- "Rename variable 'x' to 'user_count'"
- "Change function name from foo to bar"
- "Extract this code into a function"

**Classification**:
```python
{
    "route": "LOCAL_FAST",
    "confidence": 0.85,
    "reasoning": "Simple AST transformation"
}
```

**Escalation trigger**: Complex scope analysis needed

### 5. Documentation Lookup

**Query patterns**:
- "What does the Router.background config do?"
- "How do I configure transformers?"
- "Show me an example of MLX integration"

**Classification**:
```python
{
    "route": "LOCAL_MEDIUM",
    "confidence": 0.90,
    "reasoning": "Information retrieval from known docs",
    "tools": ["vector_search", "grep"]
}
```

**Enhancement**: Combine with RAG over documentation

### 6. Test Generation (Simple)

**Query patterns**:
- "Write unit tests for this add() function"
- "Create tests for this simple class"
- "Test this pure function"

**Classification**:
```python
{
    "route": "LOCAL_MEDIUM",
    "confidence": 0.80,
    "reasoning": "Straightforward function, clear contract"
}
```

**Escalation to Sonnet if**:
- Complex business logic
- Many edge cases
- Integration testing
- Mocking required

### 7. Background Research

**Query patterns**:
- "Search codebase for all uses of SessionCache"
- "Find all API endpoints"
- "List all classes that inherit from BaseModel"

**Classification**:
```python
{
    "route": "TOOL_ONLY",
    "confidence": 0.99,
    "reasoning": "Grep/ripgrep handles this perfectly"
}
```

### 8. Syntax Error Fixing

**Query patterns**:
- "Fix this syntax error: missing closing brace"
- "Fix indentation error on line 42"
- "Close unclosed string"

**Classification**:
```python
{
    "route": "LOCAL_TINY",
    "confidence": 0.95,
    "reasoning": "Compiler errors are deterministic"
}
```

**Speed**: 10x faster than Sonnet, uses smallest model

### 9. Import Organization

**Query patterns**:
- "Organize imports in this file"
- "Sort imports alphabetically"
- "Remove unused imports"

**Classification**:
```python
{
    "route": "TOOL_ONLY",
    "confidence": 0.99,
    "reasoning": "isort/ruff handles this"
}
```

### 10. Simple Questions

**Query patterns**:
- "What's in this directory?"
- "How many files are in src/?"
- "What Python version is this project using?"

**Classification**:
```python
{
    "route": "CLOUD_HAIKU" or "LOCAL_FAST",
    "confidence": 0.90,
    "reasoning": "Simple ls + summary, Haiku is 25x cheaper"
}
```

## Implementation Approaches

### Approach 1: Heuristic-Based (Simple, Immediate)

```python
class HeuristicPreRouter:
    """Rule-based pre-routing using keyword matching."""

    def __init__(self):
        self.local_keywords = {
            'list', 'find', 'search', 'grep', 'count', 'rename',
            'format', 'lint', 'fix syntax', 'organize imports',
            'commit', 'git', 'add', 'remove', 'delete', 'show'
        }

        self.tool_only_keywords = {
            'search codebase', 'find all', 'list files',
            'show directory', 'git status', 'organize imports'
        }

        self.sonnet_keywords = {
            'design', 'architect', 'explain why', 'debug complex',
            'optimize algorithm', 'security review', 'refactor complex',
            'implement feature', 'write specification'
        }

    def classify(
        self,
        query: str,
        context_size: int = 0,
        files_modified: int = 0
    ) -> RouteDecision:
        """Classify task based on heuristics."""

        query_lower = query.lower()

        # Check for tool-only patterns
        if self._matches_tool_only(query_lower):
            return RouteDecision.TOOL_ONLY

        # Check for local routing
        if self._matches_local(query_lower, files_modified):
            return self._select_local_tier(query_lower, context_size)

        # Check for SOTA requirements
        if self._requires_sonnet(query_lower, context_size):
            return RouteDecision.CLOUD_SONNET

        # Default to medium local model
        return RouteDecision.LOCAL_MEDIUM

    def _matches_tool_only(self, query: str) -> bool:
        """Check if query can be handled by tools alone."""
        return any(kw in query for kw in self.tool_only_keywords)

    def _matches_local(self, query: str, files_modified: int) -> bool:
        """Check if query is suitable for local model."""
        if any(kw in query for kw in self.local_keywords):
            if files_modified < 5:  # Simple scope
                return True
        return False

    def _select_local_tier(self, query: str, context_size: int) -> RouteDecision:
        """Select appropriate local model size."""

        # Tiny models for trivial tasks
        if 'syntax' in query or 'fix error' in query:
            return RouteDecision.LOCAL_TINY

        # Large models for complex generation
        if context_size > 10000 or 'generate' in query:
            return RouteDecision.LOCAL_LARGE

        # Medium for everything else
        return RouteDecision.LOCAL_FAST

    def _requires_sonnet(self, query: str, context_size: int) -> bool:
        """Check if Sonnet is required."""

        # Keyword match
        if any(kw in query for kw in self.sonnet_keywords):
            return True

        # Large context requires powerful model
        if context_size > 20000:
            return True

        # Long query = complex task
        if len(query) > 1000:
            return True

        return False
```

### Approach 2: ML-Based (Better Accuracy)

```python
class MLPreRouter:
    """ML model-based pre-routing."""

    def __init__(self, model_path: str = "qwen2.5-1.5b-router"):
        """
        Load a fine-tuned classification model.

        Model should be fine-tuned on routing decisions from
        your actual usage patterns.
        """
        self.model, self.tokenizer = mlx_lm.load(model_path)

        # Prompt template for classification
        self.prompt_template = """<|system|>
You are a routing classifier. Analyze the user's query and classify it into one of these categories:
- TOOL_ONLY: Can be handled by direct tools (grep, ls, git)
- LOCAL_TINY: Simple syntax fixes (1-2B model)
- LOCAL_FAST: File operations, git, formatting (7B model)
- LOCAL_MEDIUM: Simple refactoring, tests (14B model)
- LOCAL_LARGE: Complex code generation (30B model)
- CLOUD_HAIKU: Simple questions, summaries (cheap cloud)
- CLOUD_SONNET: Complex reasoning, architecture (expensive cloud)

Respond with ONLY the category name and confidence (0-1).

<|user|>
Query: {query}
Context size: {context_size} tokens
Files modified: {files_modified}

<|assistant|>
Category: """

    async def classify(
        self,
        query: str,
        context_size: int = 0,
        files_modified: int = 0,
        recent_context: list = None
    ) -> tuple[RouteDecision, float]:
        """
        Classify using ML model.

        Returns:
            (route_decision, confidence)
        """

        # Build prompt
        prompt = self.prompt_template.format(
            query=query,
            context_size=context_size,
            files_modified=files_modified
        )

        # Generate classification (very fast, <100ms)
        response = mlx_lm.generate(
            self.model,
            self.tokenizer,
            prompt=prompt,
            max_tokens=20,  # Only need category + confidence
            temp=0.1  # Low temperature for consistent classification
        )

        # Parse response
        route, confidence = self._parse_classification(response)

        return route, confidence

    def _parse_classification(self, response: str) -> tuple[RouteDecision, float]:
        """Parse model output to extract route and confidence."""

        # Expected format: "LOCAL_FAST 0.95"
        parts = response.strip().split()

        try:
            category = parts[0]
            confidence = float(parts[1]) if len(parts) > 1 else 0.9

            route = RouteDecision[category]
            return route, confidence

        except (KeyError, ValueError):
            # Fallback to safe default
            return RouteDecision.LOCAL_MEDIUM, 0.5
```

### Approach 3: Hybrid (Recommended)

```python
class HybridPreRouter:
    """Combines heuristics and ML for best results."""

    def __init__(self):
        self.heuristic_router = HeuristicPreRouter()
        self.ml_router = MLPreRouter()

        # Use heuristics for fast-path decisions
        self.fast_path_patterns = {
            'git status', 'list files', 'show directory',
            'format code', 'organize imports'
        }

    async def classify(
        self,
        query: str,
        context_size: int = 0,
        files_modified: int = 0
    ) -> tuple[RouteDecision, float]:
        """
        Classify using hybrid approach.

        Fast path: Heuristics for obvious cases (5-10ms)
        Slow path: ML model for complex decisions (50-100ms)
        """

        query_lower = query.lower()

        # Fast path: Obvious cases
        if any(pattern in query_lower for pattern in self.fast_path_patterns):
            route = self.heuristic_router.classify(query, context_size, files_modified)
            return route, 0.99  # High confidence in heuristics for these

        # Slow path: Use ML for nuanced decisions
        route, confidence = await self.ml_router.classify(
            query, context_size, files_modified
        )

        # Validate with heuristics (safety check)
        heuristic_route = self.heuristic_router.classify(query, context_size, files_modified)

        # If disagreement and low confidence, prefer heuristic
        if route != heuristic_route and confidence < 0.8:
            return heuristic_route, 0.7

        return route, confidence
```

## The Context Problem

### Problem Statement

When routing different requests to different models, context can be lost:

```
User: "List all Python files"
Router: → Local 7B model
Response: "Found 47 files: api.py, server.py, ..."

User: "Now analyze the architecture of those files"
Router: → Sonnet (complex analysis needed)
Problem: Sonnet doesn't know about the 47 files!
```

### Solution 1: Context Forwarding

**Simple approach**: Always include previous exchange summary

```python
class SimpleContextForwarder:
    """Include summary of previous work when switching models."""

    async def route_with_context(
        self,
        session_id: str,
        query: str,
        messages: list
    ) -> tuple[RouteDecision, list]:
        """Route and inject context if model changes."""

        # Get routing decision
        route = await self.pre_router.classify(query)

        # Check if model is changing
        last_model = self.get_last_model(session_id)

        if route != last_model and len(messages) > 0:
            # Inject context summary
            summary = self._summarize_recent_work(messages[-3:])

            messages.insert(0, {
                "role": "system",
                "content": f"[Previous work context]: {summary}"
            })

        return route, messages

    def _summarize_recent_work(self, messages: list) -> str:
        """Create compact summary of recent messages."""

        summaries = []
        for msg in messages:
            if msg["role"] == "assistant":
                # Truncate long responses
                content = msg["content"][:200]
                summaries.append(f"- {content}...")

        return "\n".join(summaries)
```

### Solution 2: Background Context Sync (Better)

**Maintain parallel context in SOTA model**:

```python
class BackgroundContextSync:
    """Keeps Sonnet's context updated in background."""

    def __init__(self):
        self.shadow_contexts = {}  # session_id -> context_queue
        self.sync_tasks = {}

    async def handle_local_completion(
        self,
        session_id: str,
        query: str,
        response: str,
        route: RouteDecision
    ):
        """
        When local model completes, sync to Sonnet.

        This runs in background, doesn't block user.
        """

        # Build compact summary
        summary = {
            "timestamp": time.time(),
            "query": query,
            "response_summary": response[:300],  # First 300 chars
            "route_used": route.value,
            "key_points": self._extract_key_points(response)
        }

        # Add to shadow context
        if session_id not in self.shadow_contexts:
            self.shadow_contexts[session_id] = []

        self.shadow_contexts[session_id].append(summary)

        # Optionally: Prime Sonnet with context
        # (costs tokens but keeps context warm)
        if self._should_prime(session_id):
            asyncio.create_task(
                self.prime_sonnet_context(session_id)
            )

    async def prime_sonnet_context(self, session_id: str):
        """
        Make a cheap call to Sonnet to prime its context.

        Use system message to inject recent work without
        requiring a response.
        """

        summaries = self.shadow_contexts[session_id][-5:]  # Last 5 actions
        context_text = self._format_summaries(summaries)

        # Make minimal API call (no response needed)
        await call_sonnet(
            messages=[{
                "role": "user",
                "content": "Context update (no response needed)"
            }],
            system=[{
                "type": "text",
                "text": f"[Recent work context]:\n{context_text}"
            }],
            max_tokens=1  # Minimal cost
        )

    def _should_prime(self, session_id: str) -> bool:
        """Decide if we should prime Sonnet's context."""

        # Prime after 3 local completions
        local_count = len(self.shadow_contexts.get(session_id, []))
        return local_count > 0 and local_count % 3 == 0

    def _extract_key_points(self, response: str) -> list[str]:
        """Extract key information from response."""

        key_points = []

        # Extract file names
        if "files:" in response.lower():
            # ... extract file list
            pass

        # Extract numbers/counts
        numbers = re.findall(r'\d+', response)
        if numbers:
            key_points.append(f"counts: {numbers}")

        # Extract action taken
        actions = ['created', 'modified', 'deleted', 'found', 'analyzed']
        for action in actions:
            if action in response.lower():
                key_points.append(f"action: {action}")

        return key_points
```

### Solution 3: Unified Transcript (Recommended)

**Maintain single source of truth, with smart compression**:

```python
class UnifiedTranscript:
    """
    Maintains unified transcript across all models.

    Compresses old local work into summaries for SOTA model.
    """

    def __init__(self):
        self.transcripts = {}  # session_id -> full transcript

    async def route_request(
        self,
        session_id: str,
        query: str
    ) -> str:
        """
        Route request with intelligent context management.
        """

        # Get transcript
        transcript = self.transcripts.get(session_id, [])

        # Classify task
        route = await self.pre_router.classify(query, transcript)

        # Route to appropriate model with appropriate context
        if route in [RouteDecision.LOCAL_TINY, RouteDecision.LOCAL_FAST]:
            # Use compressed context for local models
            response = await self._call_local_model(
                route=route,
                query=query,
                context=self._compress_for_local(transcript)
            )

        elif route == RouteDecision.CLOUD_SONNET:
            # Use enhanced context for Sonnet
            response = await self._call_sonnet(
                query=query,
                context=self._enhance_for_sonnet(transcript)
            )

        else:  # Haiku, etc.
            response = await self._call_cloud_model(
                route=route,
                query=query,
                context=transcript
            )

        # Add to transcript with metadata
        transcript.append({
            "role": "assistant",
            "content": response,
            "model": route.value,
            "summary": self._summarize(response),
            "timestamp": time.time()
        })

        self.transcripts[session_id] = transcript

        return response

    def _compress_for_local(self, transcript: list) -> list:
        """
        Compress transcript for local model.

        Local models don't need full history, just recent context.
        """

        # Keep only last 3 exchanges
        recent = transcript[-6:]  # 3 user + 3 assistant

        # Compress older work into summary
        if len(transcript) > 6:
            older = transcript[:-6]
            summary = self._create_summary(older)

            return [{
                "role": "system",
                "content": f"[Previous work]: {summary}"
            }] + recent

        return recent

    def _enhance_for_sonnet(self, transcript: list) -> list:
        """
        Prepare transcript for Sonnet.

        Replace verbose local model outputs with summaries.
        Keep Sonnet's own outputs intact.
        """

        enhanced = []

        for msg in transcript:
            model = msg.get("model", "")

            if "local" in model:
                # Replace verbose local output with summary
                enhanced.append({
                    "role": msg["role"],
                    "content": f"[Previous work by local model: {msg['summary']}]"
                })
            else:
                # Keep Sonnet's messages as-is
                enhanced.append({
                    "role": msg["role"],
                    "content": msg["content"]
                })

        return enhanced

    def _summarize(self, response: str) -> str:
        """
        Create compact summary of response.

        For injecting into future Sonnet context.
        """

        # Use local tiny model to summarize (very fast)
        summary = self.tiny_model.generate(
            f"Summarize in one sentence: {response[:500]}"
        )

        return summary

    def _create_summary(self, messages: list) -> str:
        """Create summary of multiple messages."""

        summaries = [msg.get("summary", msg["content"][:100]) for msg in messages]
        return " → ".join(summaries)
```

## Cost-Benefit Analysis

### Example Workflow: 10 Requests/Day

#### Current Approach (All Sonnet 4.5)

```
10 requests × 1000 tokens average = 10,000 tokens/day
Cost: 10,000 × $0.003 = $0.03/day = $11/year

Time: 10 requests × 3 seconds avg = 30 seconds/day
```

#### With Intelligent Pre-Routing

```
Distribution:
- 6 requests → LOCAL (file ops, git, formatting)
- 3 requests → HAIKU (simple questions)
- 1 request → SONNET (complex architecture)

Costs:
- 6 × LOCAL = $0 (free)
- 3 × HAIKU × 1000 tokens × $0.00025 = $0.00075
- 1 × SONNET × 1000 tokens × $0.003 = $0.003

Total: $0.00375/day = $1.37/year

Savings: 89% cost reduction ($9.63/year saved)

Time:
- 6 × LOCAL × 0.5 sec = 3 seconds
- 3 × HAIKU × 2 sec = 6 seconds
- 1 × SONNET × 3 sec = 3 seconds
Total: 12 seconds/day

Time savings: 60% faster (18 seconds saved/day)
```

### Overhead of Pre-Router

```
Classification: 50-100ms per request
10 requests × 75ms = 0.75 seconds/day

Cost: Free (local model)

Net benefit: Still 60% faster overall
```

## Integration with Existing Router

### Middleware Architecture

```python
class PreRouterMiddleware:
    """
    Middleware that runs before existing router.

    Intercepts requests and decides if they need pre-routing.
    """

    def __init__(self, app, config):
        super().__init__(app)
        self.config = config
        self.pre_router = HybridPreRouter()
        self.context_manager = UnifiedTranscript()

        # Statistics
        self.stats = {
            "total_requests": 0,
            "pre_routed": 0,
            "token_savings": 0,
            "time_savings": 0
        }

    async def dispatch(self, request: Request, call_next):
        """Pre-route requests if enabled."""

        # Check if pre-routing is enabled
        if not self.config.get("ENABLE_PRE_ROUTING", False):
            return await call_next(request)

        # Only pre-route /v1/messages
        if not request.url.path.startswith("/v1/messages"):
            return await call_next(request)

        # Get request body
        body = await request.json()
        query = self._extract_query(body)
        session_id = self._extract_session_id(body)

        # Classify with pre-router
        start_time = time.time()
        route, confidence = await self.pre_router.classify(
            query=query,
            context_size=self._estimate_context_size(body),
            files_modified=self._count_modified_files(body)
        )
        classification_time = time.time() - start_time

        # Update request with pre-routing decision
        request.state.pre_route = route
        request.state.pre_route_confidence = confidence
        request.state.classification_time = classification_time

        # Update statistics
        self._update_stats(route, confidence, classification_time)

        # Log decision
        request.app.state.logger.info(
            f"Pre-router: {route.value} (confidence: {confidence:.2f}, "
            f"time: {classification_time*1000:.0f}ms)"
        )

        # Continue to main router
        return await call_next(request)
```

### Configuration

```json5
{
  // Enable pre-routing
  "ENABLE_PRE_ROUTING": true,

  // Pre-router model
  "PRE_ROUTER_MODEL": "qwen2.5-1.5b-instruct",

  // Confidence threshold for auto-routing
  "PRE_ROUTER_CONFIDENCE_THRESHOLD": 0.8,

  // Fallback if confidence is low
  "PRE_ROUTER_FALLBACK": "LOCAL_MEDIUM",

  // Enable context syncing
  "PRE_ROUTER_CONTEXT_SYNC": true,

  // Statistics logging
  "PRE_ROUTER_LOG_STATS": true,

  // Route mappings
  "PRE_ROUTER_ROUTES": {
    "TOOL_ONLY": null,  // Direct tool execution
    "LOCAL_TINY": "mlx,qwen2.5-1.5b",
    "LOCAL_FAST": "mlx,qwen2.5-coder-7b",
    "LOCAL_MEDIUM": "mlx,qwen2.5-coder-14b",
    "LOCAL_LARGE": "mlx,qwen2.5-coder-32b",
    "CLOUD_HAIKU": "openrouter,claude-3-5-haiku",
    "CLOUD_SONNET": "openrouter,claude-3-5-sonnet",
    "CLOUD_OPUS": "openrouter,claude-opus-4"
  }
}
```

## Performance Monitoring

### Metrics to Track

```python
class PreRouterMetrics:
    """Track pre-router performance."""

    def __init__(self):
        self.metrics = {
            "total_requests": 0,
            "route_distribution": defaultdict(int),
            "confidence_distribution": [],
            "classification_times": [],
            "token_savings": 0,
            "cost_savings": 0.0,
            "time_savings": 0.0,
            "routing_accuracy": []  # User feedback
        }

    def record_routing(
        self,
        route: RouteDecision,
        confidence: float,
        classification_time: float,
        tokens_saved: int = 0
    ):
        """Record a routing decision."""

        self.metrics["total_requests"] += 1
        self.metrics["route_distribution"][route.value] += 1
        self.metrics["confidence_distribution"].append(confidence)
        self.metrics["classification_times"].append(classification_time)
        self.metrics["token_savings"] += tokens_saved

        # Calculate cost savings
        if route.value.startswith("LOCAL"):
            # Saved Sonnet tokens
            self.metrics["cost_savings"] += tokens_saved * 0.003

    def get_stats(self) -> dict:
        """Get summary statistics."""

        return {
            "total_requests": self.metrics["total_requests"],
            "routes": dict(self.metrics["route_distribution"]),
            "avg_confidence": np.mean(self.metrics["confidence_distribution"]),
            "avg_classification_time_ms": np.mean(self.metrics["classification_times"]) * 1000,
            "total_tokens_saved": self.metrics["token_savings"],
            "total_cost_savings": self.metrics["cost_savings"],
            "local_routing_rate": self._calculate_local_rate()
        }

    def _calculate_local_rate(self) -> float:
        """Calculate percentage of requests routed locally."""

        local_routes = sum(
            count for route, count in self.metrics["route_distribution"].items()
            if route.startswith("LOCAL") or route == "TOOL_ONLY"
        )

        return local_routes / max(self.metrics["total_requests"], 1)
```

### Dashboard Endpoint

```python
@app.get("/api/pre-router/stats")
async def get_pre_router_stats(request: Request):
    """Get pre-router statistics."""

    metrics = request.app.state.pre_router_metrics
    return metrics.get_stats()
```

Example output:
```json
{
  "total_requests": 1247,
  "routes": {
    "LOCAL_FAST": 612,
    "LOCAL_MEDIUM": 298,
    "TOOL_ONLY": 156,
    "CLOUD_HAIKU": 87,
    "CLOUD_SONNET": 94
  },
  "avg_confidence": 0.89,
  "avg_classification_time_ms": 67,
  "total_tokens_saved": 456789,
  "total_cost_savings": 1.37,
  "local_routing_rate": 0.85
}
```

## Future Enhancements

### 1. Adaptive Learning

Learn from user corrections:

```python
@app.post("/api/pre-router/feedback")
async def record_routing_feedback(
    session_id: str,
    was_correct: bool,
    should_have_been: str
):
    """
    User can provide feedback on routing decisions.

    Use this to fine-tune the pre-router model.
    """
    # Store feedback for model retraining
    pass
```

### 2. User-Specific Routing

Learn user preferences:

```python
class PersonalizedPreRouter:
    """Learn individual user routing preferences."""

    def __init__(self):
        self.user_preferences = {}  # user_id -> preferences

    def classify(self, user_id: str, query: str):
        """Classify with user-specific adjustments."""

        # Get base classification
        route, confidence = self.base_classifier.classify(query)

        # Adjust based on user preferences
        prefs = self.user_preferences.get(user_id, {})

        if prefs.get("prefers_local", False):
            # User prefers local models, lower Sonnet threshold
            if route == RouteDecision.CLOUD_SONNET and confidence < 0.9:
                route = RouteDecision.LOCAL_LARGE

        return route, confidence
```

### 3. Cost-Aware Routing

Factor in current budget:

```python
class CostAwarePreRouter:
    """Route based on remaining budget."""

    def __init__(self, daily_budget: float = 1.0):
        self.daily_budget = daily_budget
        self.spent_today = 0.0

    def classify(self, query: str):
        """Classify with budget awareness."""

        route, confidence = self.base_classifier.classify(query)

        # If budget is low, prefer local models
        remaining = self.daily_budget - self.spent_today

        if remaining < 0.10:  # Less than 10 cents left
            if route == RouteDecision.CLOUD_SONNET:
                # Downgrade to local or Haiku
                route = RouteDecision.LOCAL_LARGE if confidence < 0.95 else RouteDecision.CLOUD_HAIKU

        return route, confidence
```

## Implementation Timeline

### Phase 1: Heuristic Router (1-2 days)
- Implement keyword-based routing
- Basic route categories
- Simple context forwarding
- Test with common queries

### Phase 2: Context Management (2-3 days)
- Implement unified transcript
- Background context sync
- Context compression for local models
- Context enhancement for Sonnet

### Phase 3: ML-Based Router (3-5 days)
- Fine-tune 1.5B model on routing decisions
- Implement ML classification
- Hybrid approach (heuristics + ML)
- Performance optimization

### Phase 4: Monitoring & Tuning (2-3 days)
- Metrics collection
- Dashboard for statistics
- User feedback mechanism
- Model refinement

**Total: 8-13 days for full implementation**

## Conclusion

Intelligent pre-routing with a small, fast model can dramatically improve both cost efficiency and user experience. The combination of instant local routing for simple tasks and powerful cloud models for complex work provides the best of both worlds.

Key benefits:
- **89% cost reduction** through smart routing
- **60% faster** average response time
- **Better resource utilization** (right tool for the job)
- **Seamless experience** with maintained context

The overhead of classification (50-100ms) is negligible compared to the benefits, and the system can be implemented incrementally, starting with simple heuristics and evolving to ML-based classification.
