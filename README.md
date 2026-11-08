# 🔗 nanomcp Stack

Self-hosted deployment of nanomcp — a Model Context Protocol gateway written
in Zig: one binary that proxies MCP clients to upstream MCP servers (stdio /
SSE / streamable-HTTP) with inbound token/OAuth 2.1 auth, plus a stub
streamable-HTTP upstream for the functional gate.

## 🧩 Components

### 🖥️ [nanomcp App](app/)

The `nanomcp` gateway service (source-built Zig image, no published upstream
image) with the `forwarding` environment and the `stub-upstream` gate
companion; the gateway binds `127.0.0.1:7331` in-container and is exposed
through a socat relay on port 7332.

## 🚀 Quick Start

```bash
cd app
docker build -t localhost/nanomcp:0.1.0 images/nanomcp   # source build, long
sb build
cd build
cp .env.example .env          # set NANOMCP_SEED_INBOUND_TOKEN
docker compose up -d
```

See [`app/README.md`](app/README.md) for the full runbook.

## 📋 Requirements

- Docker & Docker Compose (BuildKit for the source image build)
- 4+ GB free RAM for the image build (Zig compile; 10–20 min)
- A strong inbound token (`NANOMCP_SEED_INBOUND_TOKEN`) for token auth

## 📄 License

This project is dual-licensed under:

- [Apache License 2.0](LICENSE-APACHE)
- [MIT License](LICENSE-MIT)
