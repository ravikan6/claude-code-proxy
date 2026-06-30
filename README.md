# Claude Code Proxy

A production-oriented Rust gateway that exposes Anthropic's Messages API to Claude Code and routes requests to OpenAI-compatible or Azure Chat Completions providers.

The Rust implementation in `rust-proxy/` is primary. The Python implementation remains temporarily as a deprecated behavioral reference.

## Capabilities

- Anthropic `/v1/messages` and `/v1/messages/count_tokens`
- Correct Anthropic SSE lifecycle, text deltas, parallel tool calls, and final usage
- Text, system prompts, base64/URL images, tools, and tool results
- Declarative model routes with priorities, weights, capability filtering, and safe fallback
- OpenAI-compatible and Azure unified-v1 endpoints
- Named client keys, per-client rate/concurrency limits, and route permissions
- Timeouts, cancellation, passive circuit breakers, load shedding, and graceful shutdown
- Atomic SIGHUP configuration reload
- Structured logs and Prometheus metrics on a separate listener
- Strict rejection when a requested feature cannot be represented safely

## Quick start

```bash
cp rust-proxy/config.example.yaml rust-proxy/config.yaml
export OPENAI_API_KEY='...'
export PROXY_CLIENT_KEY='use-a-long-random-value'

cd rust-proxy
cargo run --release -- serve --config config.yaml
```

Configure Claude Code:

```bash
ANTHROPIC_BASE_URL=http://localhost:8082 \
ANTHROPIC_API_KEY="$PROXY_CLIENT_KEY" \
claude
```

Or run with Docker:

```bash
cp rust-proxy/config.example.yaml rust-proxy/config.yaml
docker compose up --build
```

## Configuration and operations

Configuration is YAML-only. Secret values use `{ env: NAME }` or `{ file: /path }` references. The full schema and routing example are in [`rust-proxy/config.example.yaml`](rust-proxy/config.example.yaml).

```bash
claude-code-proxy check-config --config config.yaml
claude-code-proxy print-effective-config --config config.yaml
kill -HUP <pid> # atomic reload; invalid updates retain the active config
```

Health endpoints are `/health/live`, `/health/ready`, and `/health`. Prometheus metrics are exposed at `/metrics` on `metrics_bind` (default `127.0.0.1:9090`). TLS should terminate at your ingress or service mesh.

Every Messages request must include `anthropic-version: 2023-06-01` and either `x-api-key` or `Authorization: Bearer`.

## Compatibility policy

The proxy never silently discards semantic options. Extended thinking, documents/PDFs, citations, Anthropic server tools, MCP blocks, and unsupported beta features return a clear capability error for Chat Completions routes. `cache_control` is accepted as a non-semantic hint and discarded with a metric.

Token counts use the selected target's configured tokenizer and therefore describe the upstream prompt, not Anthropic's tokenizer.

## Development

```bash
cd rust-proxy
cargo fmt --check
cargo test --all-targets
cargo clippy --all-targets --all-features -- -D warnings
```

See [`rust-proxy/README.md`](rust-proxy/README.md) for architecture, failure semantics, and provider configuration details.
