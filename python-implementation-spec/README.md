# Claude Code Router - Python Implementation Specification

## Overview

This directory contains comprehensive specifications for reimplementing the Claude Code Router in Python, with a focus on integrating local MLX-based models on Apple Silicon Macs.

## Document Structure

The specification is divided into 13 documents, each covering a specific aspect of the system:

### Core System Documentation

1. **[00-overview.md](00-overview.md)** - System architecture and high-level design
   - Executive summary
   - Component responsibilities
   - Data flow diagrams
   - File system layout
   - Technology stack overview

2. **[01-configuration-system.md](01-configuration-system.md)** - Configuration management
   - JSON5 configuration format
   - Environment variable interpolation
   - Configuration validation
   - Backup and versioning
   - Pydantic data models

3. **[02-routing-engine.md](02-routing-engine.md)** - Routing logic and model selection
   - Token counting algorithm
   - Session usage tracking
   - Model selection rules (long context, background, thinking, etc.)
   - Custom router support
   - System prompt rewriting

4. **[03-server-architecture.md](03-server-architecture.md)** - HTTP server and API endpoints
   - FastAPI/Starlette server setup
   - All API endpoints specification
   - Middleware/hooks system
   - Request/response flow
   - Error handling

5. **[04-llm-providers-transformers.md](04-llm-providers-transformers.md)** - Provider integration
   - Provider abstraction
   - Transformer system (request/response adaptation)
   - Built-in transformers (Anthropic, DeepSeek, Gemini, OpenRouter, etc.)
   - Transformer chaining
   - Custom transformer support

6. **[05-agents-system.md](05-agents-system.md)** - Agent architecture
   - Agent interface specification
   - Image Agent implementation (vision capabilities for non-vision models)
   - Agent stream processing
   - Tool injection and execution
   - LRU caching for images

7. **[06-cli-lifecycle.md](06-cli-lifecycle.md)** - CLI and process management
   - All CLI commands (start, stop, restart, status, code, model, ui, statusline)
   - PID file management
   - Process checking
   - Reference counting
   - Service lifecycle

8. **[07-streaming-sse.md](07-streaming-sse.md)** - Server-Sent Events and streaming
   - SSE format specification
   - SSE parser implementation
   - SSE serializer implementation
   - Stream rewriting/transformation
   - Usage tracking via stream tee-ing

9. **[08-authentication-security.md](08-authentication-security.md)** - Security features
   - API key authentication
   - Host binding security
   - Environment variable security
   - HTTPS/TLS recommendations
   - Rate limiting strategies

10. **[09-logging-monitoring.md](09-logging-monitoring.md)** - Logging and monitoring
    - Rotating file logging
    - Log cleanup
    - Request logging middleware
    - Status line system (terminal status display)
    - Health check endpoints

### Implementation Guides

11. **[10-python-implementation.md](10-python-implementation.md)** - Python-specific implementation guide
    - Recommended technology stack (FastAPI, uvicorn, httpx, etc.)
    - Project structure
    - Complete implementation examples
    - Testing strategies
    - Deployment options (Docker, systemd)

12. **[11-mlx-integration.md](11-mlx-integration.md)** - MLX local model integration guide
    - MLX setup and installation
    - Model downloading and management
    - MLX transformer implementation
    - Internal MLX model server
    - Performance optimization
    - Hybrid local/cloud workflow recommendations

### Future Enhancements

13. **[12-intelligent-pre-routing.md](12-intelligent-pre-routing.md)** - Intelligent pre-routing with small fast models
    - Using 1B-3B models for request classification
    - 10 high-value routing use cases
    - Three implementation approaches (heuristic, ML, hybrid)
    - Context management across model switches
    - Cost-benefit analysis (89% savings, 60% faster)
    - Performance monitoring and metrics
    - Integration with existing router

## Reading Guide

### For Complete Implementation

Read the documents in order (00-12) for a comprehensive understanding of the entire system.

### For Specific Features

- **Just want to understand routing?** → Read 02
- **Need to implement transformers?** → Read 04
- **Setting up authentication?** → Read 08
- **Integrating MLX models?** → Read 11
- **Building the CLI?** → Read 06
- **Want intelligent cost-saving routing?** → Read 12

### Quick Start Path

1. Read **00-overview.md** for system architecture
2. Read **10-python-implementation.md** for implementation approach
3. Read **11-mlx-integration.md** for MLX-specific integration
4. Refer to other documents as needed for specific components

## Key Features of Python Implementation

### Advantages Over TypeScript Version

1. **Native MLX Support**: Direct integration with Apple's MLX framework
2. **Type Safety**: Pydantic models for configuration and data validation
3. **Async Performance**: Comparable to Node.js with asyncio
4. **Rich Ecosystem**: Access to Python ML/AI libraries
5. **Simpler Deployment**: Single executable with PyInstaller or docker

### MLX Integration Benefits

- **No External Dependencies**: Models run directly in the Python process
- **Unified Memory**: Leverages Apple Silicon's architecture
- **Cost Effective**: Free local inference for routine tasks
- **Privacy**: All data stays on your machine
- **Flexible**: Mix local and cloud models based on task complexity

## Implementation Roadmap

### Phase 1: Core Infrastructure
- [ ] Configuration system (01)
- [ ] Basic FastAPI server (03)
- [ ] CLI commands (06)
- [ ] Process management

### Phase 2: Routing and Providers
- [ ] Token counting (02)
- [ ] Routing engine (02)
- [ ] Basic transformers (04)
- [ ] HTTP client for providers (04)

### Phase 3: Streaming and Advanced Features
- [ ] SSE parsing/serialization (07)
- [ ] Stream processing (07)
- [ ] Authentication (08)
- [ ] Logging (09)

### Phase 4: MLX Integration
- [ ] MLX transformer (11)
- [ ] MLX model server (11)
- [ ] Model management
- [ ] Performance optimization

### Phase 5: Advanced Features
- [ ] Agent system (05)
- [ ] Image agent (05)
- [ ] Status line (09)
- [ ] Web UI

## Testing Strategy

Each component should have:
- **Unit tests**: Test individual functions and classes
- **Integration tests**: Test component interactions
- **End-to-end tests**: Test complete request flows
- **Performance tests**: Ensure streaming efficiency

See **10-python-implementation.md** for testing examples.

## Recommended Development Approach

### 1. Start Simple

Begin with a minimal implementation:
- Basic configuration loading
- Simple FastAPI server
- Single transformer (pass-through)
- Basic routing (default model only)

### 2. Iterate

Add features incrementally:
- More routing rules
- Additional transformers
- Streaming support
- MLX integration

### 3. Test Continuously

Write tests as you build:
- Use pytest for unit tests
- Use TestClient for API tests
- Test with real Claude Code

### 4. Optimize Later

Focus on correctness first:
- Get it working
- Make it correct
- Make it fast

## Dependencies Summary

### Core Dependencies
```
fastapi>=0.100.0        # Web framework
uvicorn>=0.23.0         # ASGI server
httpx>=0.24.0           # HTTP client
pydantic>=2.0.0         # Data validation
click>=8.0.0            # CLI framework
tiktoken>=0.5.0         # Token counting
pyjson5>=1.6.0          # JSON5 parsing
```

### MLX Dependencies (Optional)
```
mlx>=0.4.0              # Apple MLX framework
mlx-lm>=0.4.0           # MLX language models
huggingface_hub         # Model downloading
```

### Development Dependencies
```
pytest>=7.0.0           # Testing
pytest-asyncio          # Async testing
black                   # Code formatting
ruff                    # Linting
mypy                    # Type checking
```

## Contributing to Specification

These specifications are designed to be:
- **Complete**: All information needed for implementation
- **Clear**: Detailed explanations and examples
- **Correct**: Accurate representation of the system
- **Current**: Updated to reflect latest version

If you find gaps or errors, please update the relevant document.

## License

These specifications are derived from the Claude Code Router project and are intended for implementation purposes. Refer to the main project LICENSE for terms.

## Next Steps

1. **Read the overview** (00-overview.md) to understand the system architecture
2. **Review the Python implementation guide** (10-python-implementation.md) for technology choices
3. **Study the MLX integration guide** (11-mlx-integration.md) for local model support
4. **Start implementing** following the roadmap above
5. **Test with Claude Code** to ensure compatibility

## Questions?

These specifications are comprehensive, but if you need clarification on any aspect:
- Refer to the original TypeScript source code in `src/`
- Check the main README.md for configuration examples
- Review the example configurations in the project root

Good luck with your Python implementation! The combination of Python's ML ecosystem and MLX's performance should provide an excellent foundation for local model integration.
