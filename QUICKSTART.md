# Quick start

```bash
cp rust-proxy/config.example.yaml rust-proxy/config.yaml
export OPENAI_API_KEY='...'
export PROXY_CLIENT_KEY='long-random-key'

cd rust-proxy
cargo run --release -- serve --config config.yaml
```

Then run Claude Code through the proxy:

```bash
ANTHROPIC_BASE_URL=http://localhost:8082 \
ANTHROPIC_API_KEY="$PROXY_CLIENT_KEY" \
claude
```

Validate configuration without starting:

```bash
cargo run -- check-config --config config.yaml
```
