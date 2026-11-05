# 🔗 nanomcp App

A modular Docker Compose configuration system for the nanomcp MCP gateway —
a single Zig binary that aggregates upstream MCP servers (stdio / SSE /
streamable-HTTP) behind one HTTP surface with inbound token/OAuth auth, plus
a stub streamable-HTTP upstream used by the functional gate.

## 🚀 Quick Start

### 1. Build the Source Image

No published Docker image exists upstream, so the gateway is built from
source (Zig 0.17.1, nanoui pinned `v0.12.0`):

```bash
docker build -t localhost/nanomcp:0.1.0 images/nanomcp
```

The Dockerfile is self-contained (empty build context — it clones upstream at
the pinned commit itself). Expect a long build (Zig toolchain download +
compile).

### 2. Build Configurations

```bash
sb build
```

This generates the deployment in `build/` based on
[`stackbuilder.toml`](stackbuilder.toml). This stack produces a single merged
configuration (base + `forwarding` environment + the `entire` combo with the
stub upstream).

### 3. Configure Environment

```bash
cd build
cp .env.example .env
# Edit .env — set NANOMCP_SEED_INBOUND_TOKEN for token auth
```

### 4. Deploy

```bash
docker compose up -d
```

Access: `http://localhost:7332/mcp` (relay port; the gateway itself binds
`127.0.0.1:7331` inside the container and is reachable only through the
in-container socat relay on 7332).

## 🧩 Product Parts

The product's functional base per [`DEPLOYMENT.md`](../DEPLOYMENT.md)
(stack-conventions §20) — one compose service per part:

- **nanomcp** (gateway, single binary): the whole product — MCP gateway HTTP
  surface (`/mcp`, `/sse`, `/message`, `/healthz`), inbound auth
  (`off`/`token`/`oauth`), upstream transports, registry/market client, and
  credential store. Without it there is nothing. The nanoui desktop GUI is
  inside the same binary but is not served over HTTP; in a container the
  gateway runs behind an Xvfb virtual display with the GTK3 runtime
  (`NANOUI_GTK=3`, no real window is ever shown) and all network functionality
  is reachable by MCP clients.

Mandatory infra (database / OIDC / S3) was attempted and **refused** per
[`DEPLOYMENT.md`](../DEPLOYMENT.md): the product is file-based
(`servers.json`, credential files under `userdata/`), implements its own
OAuth 2.1 authorization server, and has no object-storage surface.

## 📁 Project Structure

- [`components/`](components/) — Source compose components
  - [`base/`](components/base/) — Base gateway configuration (service,
    volume, network, healthcheck)
  - [`environments/`](components/environments/) — Environment-specific
    configurations (`forwarding`)
  - [`extensions/`](components/extensions/) — Extension components
    (`stub-mcp-upstream`, `publish-gateway-url-discovery`)
- [`images/nanomcp/`](images/nanomcp/) — Source-built image (Dockerfile,
    empty build context)
- [`build/`](build/) — Generated configurations (auto-generated, ready to
  deploy)
- [`stackbuilder.toml`](stackbuilder.toml) — Build configuration for
  stackbuilder

## 🔧 Available Configurations

### Environments

- **forwarding**: publishes the relay port `${NANOMCP_PORT:-7332}:7332` on the
  host (the container always listens on 7332 internally).

### Extensions

- **stub-mcp-upstream** (`implemented`, `required-for-gate`): minimal
  streamable-HTTP MCP upstream (python:3-alpine, inline `command`) with one
  `echo` tool; the gateway seeds it into `servers.json` via
  `NANOMCP_SEED_STUB_URL` so the functional gate can verify
  initialize → tools/list → tools/call end to end. Status per the extension
  catalog in [`DEPLOYMENT.md`](../DEPLOYMENT.md) (stack-conventions §20).
- **mcp-gateway-stdio** (`available`): container-side stdio upstream helpers
  for real stdio upstreams — recorded, NOT implemented.
- **publish-gateway-url-discovery** (`implemented`): seeds the gateway's
  configured port from `NANOMCP_PUBLIC_PORT` so the
  `/.well-known/oauth-protected-resource` metadata advertises the
  host-published URL (`http://127.0.0.1:<port>` — upstream hardcodes the
  base to the configured port, no Host header). Inert when the env is
  empty. Status per the extension catalog in
  [`DEPLOYMENT.md`](../DEPLOYMENT.md) (stack-conventions §20).

## 🔧 Environment Variables

### Base Configuration

- `COMPOSE_PROJECT_NAME`: Project name for Docker Compose (default: nanomcp)
- `IMAGE`: Docker image (default: localhost/nanomcp:0.1.0)
- `NANOMCP_SEED_INBOUND_TOKEN`: Bearer token callers must present (inbound
  `token` mode); seeded into the credential store at container start as a
  flat 0600 file (`userdata/inbound-token.json`). Empty = inbound stays
  `off` (default: empty)
- `LOG_MAX_SIZE`: Log rotation size (default: 10m)
- `LOG_MAX_FILE`: Log rotation file count (default: 3)
- `LOG_COMPRESS`: Compress rotated logs (default: true)
- `TZ`: Timezone (default: UTC)

### Forwarding Configuration

- `NANOMCP_PORT`: Host-side published port (default: 7332; the container
  always listens on relay port 7332)

### Publish Gateway URL Discovery Configuration

- `NANOMCP_PUBLIC_PORT`: Host-published gateway port; the entrypoint seeds it
  into `servers.json` `.port` so the `/.well-known/oauth-*` metadata
  advertises `http://127.0.0.1:<port>` matching the host URL. Must equal
  `NANOMCP_PORT` and differ from the relay listen port 7332 (e.g. both 7100).
  Empty = extension inert (default: empty)

### Stub Upstream Configuration

- `NANOMCP_SEED_STUB_URL`: MCP URL the gateway seeds the stub upstream with
  (default: http://stub-mcp-upstream:8080/mcp); set to empty to skip stub
  seeding

## 🌐 Networks

- **nanomcp-network** (internal bridge): gateway + stub upstream

## 🔒 Security

⚠️ **Production Checklist:**

- Set `NANOMCP_SEED_INBOUND_TOKEN` to a strong random value — with inbound
  `off` anyone who can reach the port can use the gateway.
- Only the relay port is published; the gateway itself binds
  `127.0.0.1:7331` inside the container (hardcoded upstream) and is not
  directly reachable.
- The inbound token is stored world-unreadable (0600) in the credential
  store inside the `nanomcp-userdata` volume.
- For public deployments, put the gateway behind the collection's SSL stacks
  and consider the `publish-gateway-url-discovery` extension (OAuth
  protected-resource metadata).

## 🆘 Troubleshooting

**Build Issues:**

- `sb build` fails with "yq is required": stackbuilder needs mikefarah yq
  v4 on PATH (the distro `yq` is a different tool); see PITFALLS P-006/P-033.
- Image build fails on memory: the Zig compile needs several GB RAM; retry
  with less load (no swap/device fixes on the host).

**Gateway Issues:**

- `GET /healthz` returns 401: inbound `token` mode is active — send
  `Authorization: Bearer <NANOMCP_SEED_INBOUND_TOKEN>`.
- Container unhealthy: the healthcheck curls `http://127.0.0.1:7331/healthz`
  (accepts 200 in off mode and 401 in token mode); check `docker logs
  nanomcp` for gateway startup errors.
- Stub tool missing (`tools: 0`): check that `NANOMCP_SEED_STUB_URL` is set
  and the stub container is healthy; seeding only happens at container
  start.

## 📝 Notes

- The `build/` directory is automatically generated and should not be edited
  manually
- Environment variables in generated files use `$VARIABLE_NAME` format for
  proper interpolation
- The generated configuration is a single merged `docker-compose.yml` +
  `.env.example` (this stackbuilder version emits one `build/` directory, no
  per-env subdirs)
- Use `sb build` to regenerate the configuration after modifying components
- The gateway binds `127.0.0.1:<configured port>` (upstream hardcodes the
  address; `servers.json` holds the port, default 7331); the entrypoint runs
  a socat relay `0.0.0.0:7332 → 127.0.0.1:<configured port>` and compose
  publishes only the relay port
- Seeding (`NANOMCP_SEED_INBOUND_TOKEN`, `NANOMCP_SEED_STUB_URL`,
  `NANOMCP_PUBLIC_PORT`) never clobbers GUI-managed state: the token file is
  rewritten only when it differs, and `servers.json` entries other than the
  stub and the port are left untouched
