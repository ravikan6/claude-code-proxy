# Claude Code Proxy - Comprehensive Project Analysis

## Overview

Claude Code Proxy is a production-oriented gateway that translates Anthropic's Messages API to OpenAI-compatible and Azure Chat Completions providers. The project has two implementations:

1. **Primary (Rust)**: Located in `rust-proxy/` - A high-performance, production-ready implementation
2. **Deprecated (Python)**: Located in `src/` - A reference implementation that's being phased out

## Architecture

### Rust Implementation (Primary)

The Rust implementation follows a clean, layered architecture:

1. **Server Layer** (`server.rs`): Handles HTTP requests, authentication, and response formatting
2. **Anthropic Layer** (`anthropic.rs`): Validates and normalizes Anthropic API requests
3. **Routing Layer** (`routing.rs`): Determines which provider to use based on model patterns
4. **Provider Layer** (`provider.rs`): Interfaces with OpenAI/Azure APIs
5. **Domain Layer** (`domain.rs`): Contains data structures and business logic

#### Key Features:

- **Authentication**: Client key validation with rate limiting
- **Routing**: Model pattern matching to different providers
- **Circuit Breakers**: Failure detection and automatic recovery
- **Metrics**: Prometheus metrics endpoint
- **Configuration Reload**: SIGHUP support for atomic config reload
- **Graceful Shutdown**: SIGINT/SIGTERM handling
- **SSE Streaming**: Full support for Anthropic's streaming format

#### Data Flow:

```
Client Request → Server (auth/validation) → Anthropic (normalize) → Router (route selection) → Provider (API call) → Response
```

### Python Implementation (Deprecated)

The Python implementation uses FastAPI and provides similar functionality:

1. **Main Entry** (`main.py`): FastAPI app setup and configuration
2. **Endpoints** (`api/endpoints.py`): HTTP route handlers
3. **Client** (`core/client.py`): OpenAI API client with cancellation support
4. **Converters** (`conversion/`): Request/response format conversion
5. **Models** (`models/`): Pydantic models for request validation

#### Key Features:

- FastAPI-based web server
- Async OpenAI client with cancellation support
- Claude-to-OpenAI request/response conversion
- Streaming and non-streaming support
- Tool calling and multimodal support

## Configuration

### Rust Configuration

The Rust proxy uses YAML configuration (`config.yaml`):

```yaml
server:
  bind: "0.0.0.0:8082"
  metrics_bind: "0.0.0.0:9090"
  max_body_bytes: 16777216
  shutdown_grace_seconds: 30
  require_anthropic_version: true
  loose_input_validation: false

limits:
  global_concurrency: 512
  connect_timeout_ms: 5000
  request_timeout_seconds: 120
  stream_idle_timeout_seconds: 60
  max_attempts: 2
  circuit_failure_threshold: 5
  circuit_cooldown_seconds: 30

clients:
  - id: engineering
    key:
      env: PROXY_CLIENT_KEY
    allowed_routes: [sonnet, haiku]
    requests_per_minute: 600
    concurrent_requests: 100

providers:
  - id: openai
    kind: openai_chat
    endpoint: "https://api.openai.com/v1"
    credential:
      type: bearer
      secret:
        env: OPENAI_API_KEY
    capability_profile:
      tokenizer: o200k_base
      vision: true
      tools: true
      parallel_tools: true
      supports_temperature: true
      supports_top_p: true
      supports_stop: true
      max_output_tokens: 16384
      use_max_completion_tokens: false

routes:
  - id: sonnet
    models: ["claude-*-sonnet-*", "claude-sonnet-*"]
    targets:
      - provider: openai
        model: gpt-4.1
        priority: 1
        weight: 100
```

### Python Configuration

The Python proxy uses environment variables:

```bash
export OPENAI_API_KEY='your-openai-key'
export ANTHROPIC_API_KEY='your-anthropic-key'  # Optional for client validation
export OPENAI_BASE_URL='https://api.openai.com/v1'
export BIG_MODEL='gpt-4o'
export SMALL_MODEL='gpt-4o-mini'
export HOST='0.0.0.0'
export PORT='8082'
export LOG_LEVEL='INFO'
export MAX_TOKENS_LIMIT='4096'
export REQUEST_TIMEOUT='90'
```

## API Endpoints

Both implementations provide the following endpoints:

### Main Endpoints

- `POST /v1/messages` - Chat completion (streaming and non-streaming)
- `POST /v1/messages/count_tokens` - Token counting

### Health/Status Endpoints

- `GET /health/live` - Liveness check
- `GET /health/ready` - Readiness check
- `GET /health` - Health status
- `GET /test-connection` - Test OpenAI connectivity (Python only)
- `GET /` - Root endpoint with info

## Model Mapping

The proxy maps Claude model names to OpenAI models:

- **Haiku models** → `SMALL_MODEL` / `gpt-4o-mini`
- **Sonnet models** → `MIDDLE_MODEL` / `gpt-4o`
- **Opus models** → `BIG_MODEL` / `gpt-4o`

Pattern matching is used to identify model types:
- `claude-*-haiku-*` → Small model
- `claude-*-sonnet-*` → Middle model
- `claude-*-opus-*` → Big model

## Features

### Supported Features

1. **Text Completion**: Basic chat completion
2. **Streaming**: Full SSE streaming support
3. **System Messages**: Single string or block format
4. **Multimodal**: Text + image inputs
5. **Tool Calling**: Function/tool definitions and usage
6. **Tool Results**: Handling tool execution results
7. **Token Counting**: Input token estimation
8. **Cancellation**: Request cancellation support
9. **Authentication**: Client API key validation
10. **Rate Limiting**: Per-client rate limits
11. **Circuit Breakers**: Failure detection and recovery (Rust only)
12. **Metrics**: Prometheus metrics (Rust only)

### Unsupported Features

- Extended thinking
- Documents/PDFs
- Citations
- Anthropic server tools
- MCP blocks
- `top_k` parameter

## Testing

### Python Tests

The `tests/test_main.py` file provides comprehensive tests:

- Basic chat completion
- Streaming chat
- Function calling
- System messages
- Multimodal inputs
- Conversation with tool use
- Token counting
- Health checks

### Rust Tests

The Rust implementation includes unit tests and integration tests in the `tests/` directory.

## Deployment

### Rust Deployment

```bash
# Build and run
cp rust-proxy/config.example.yaml rust-proxy/config.yaml
export OPENAI_API_KEY='...'
export PROXY_CLIENT_KEY='use-a-long-random-value'

cd rust-proxy
cargo run --release -- serve --config config.yaml

# With Docker
cp rust-proxy/config.example.yaml rust-proxy/config.yaml
docker compose up --build
```

### Python Deployment

```bash
# Install dependencies
pip install -r requirements.txt

# Run server
export OPENAI_API_KEY='your-key'
python start_proxy.py

# With Docker
# Note: Dockerfile.python is available but docker-compose.yml is missing
```

## Usage with Claude Code

```bash
ANTHROPIC_BASE_URL=http://localhost:8082 \
ANTHROPIC_API_KEY="$PROXY_CLIENT_KEY" \
claude
```

## Key Differences: Rust vs Python

| Feature | Rust | Python |
|---------|------|--------|
| Performance | High (native) | Medium (Python async) |
| Production Ready | Yes | No (deprecated) |
| Configuration | YAML | Environment variables |
| Metrics | Prometheus | None |
| Reload | SIGHUP support | No |
| Circuit Breakers | Yes | No |
| Rate Limiting | Per-client | Basic |
| Concurrency Control | Advanced | Basic |
| SSE Streaming | Full support | Full support |
| Cancellation | Yes | Yes |
| Tool Calling | Yes | Yes |
| Multimodal | Yes | Yes |

## Recommendations

1. **Use Rust for Production**: The Rust implementation is designed for production use with better performance, reliability, and monitoring features.

2. **Python for Reference**: The Python implementation serves as a good reference for understanding the conversion logic but should not be used in production.

3. **Configuration**: Use the YAML configuration for flexibility and security (secrets from environment files).

4. **Monitoring**: Leverage the Prometheus metrics endpoint in the Rust version for observability.

5. **Scaling**: The Rust implementation supports higher concurrency and has better resource management.

## Future Work

1. Complete the deprecation of the Python implementation
2. Add more comprehensive documentation for the Rust version
3. Implement additional provider types (Ollama, etc.)
4. Enhance error handling and retry logic
5. Add more detailed metrics and tracing
6. Implement request logging (with PII redaction)
7. Add support for additional Anthropic features as they become stable
