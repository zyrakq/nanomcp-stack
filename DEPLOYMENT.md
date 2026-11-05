# DEPLOYMENT — nanomcp

Upstream: https://github.com/sternelee/nanomcp (cloned and analyzed at commit `90cffaaec6ea17f55f7cb8c13871c3f8f03a88f0`, 2026-11-05).

nanomcp is a Model Context Protocol gateway written in Zig (0.17): one binary that
runs an HTTP gateway surface for MCP clients, proxies to upstream MCP servers
(stdio / SSE / streamable-HTTP), acts as its own OAuth 2.1 authorization server
for inbound auth, and ships a **native desktop GUI on nanoui** — not a web UI.

## Product parts

| part | purpose | image source |
|------|---------|--------------|
| `nanomcp` (gateway, single binary) | MCP gateway HTTP surface (`/mcp`, `/sse`, `/message`, `/healthz`) + inbound auth (`off`/`token`/`oauth` modes; gateway as OAuth 2.1 authserver) + upstream transports (stdio, SSE, streamable-HTTP) + registry/market client (`https://registry.modelcontextprotocol.io` `GET /v0/servers`) + credentials/token store | **source-built** (no published Docker image; `sternelee/nanomcp` → 404 on Docker Hub) per §19: `app/images/nanomcp/Dockerfile`, Zig 0.17, nanoui dep pinned `v0.12.0` in `build.zig.zon` |
| nanoui GUI (inside the same binary) | native desktop window (1080×740), 5 pages: Servers / Market / Routes / Tools / Settings | not a separate part |

Facts from source (grep-verified):
- Serving auto-starts on launch: `state.App.init` (`src/state.zig`) calls `gw.startAsync(cfg.port)` — no saved serving flag.
- `GET /healthz` → `{"status":"ok","tools":N}`.
- Inbound modes (`config.zig` `InboundMode`): `.off`, `.token` (user-set bearer token), `.oauth` (bearer + gateway's own OAuth 2.1 AS: `/oauth/register`, `/oauth/authorize`, `/oauth/token`, `/oauth/callback`, `/.well-known/oauth-protected-resource[/path]`, `/.well-known/oauth-authorization-server`).
- Bind: hardcoded `127.0.0.1` in `src/server.zig` (`net.Ip4Address.loopback(port)`); **no env var for bind or port** — port is changed in the Settings page (GUI) and persisted in `servers.json` (default `7331`, inbound `off`).
- Only env var in source: `NANOMCP_TEST_DIR` (test-support). Headless/E2E nanoui helper flags: `--nanoui-screenshot`/`--nanoui-eval` (imply `--nanoui-hidden`) — nanoui harness flags, not deployment config.
- Persistence: `$HOME/.local/share/nanomcp/userdata/` — `servers.json` (routes/port/inbound), `gateway-tokens.json`, per-server OAuth token files (`oauth-token/<id>`, mode 0600; dirs 0700).

## Interaction map

```
MCP clients ──▶ nanomcp gateway (7331, MCP + OAuth endpoints only)
nanomcp gateway ──▶ upstream MCP servers (stdio child procs / SSE / streamable-HTTP)
nanoui GUI (native window, in-process) ──▶ same gateway state
nanomcp gateway ──▶ registry.modelcontextprotocol.io (outbound https, market page)
```

- Without the gateway there is nothing: it is the whole product.
- The GUI is **not served over HTTP**: `gateway_http.zig` dispatches only MCP and
  OAuth endpoints; the window is a nanoui native process (`main.zig`). An MCP
  client reaches everything the network surface offers; GUI-only operations
  (adding servers via Market, Settings port change) are window-bound.
- Absent parts: nothing breaks — single part, no infra peers.

## Functional base

| group | tier | parts | justification |
|-------|------|-------|---------------|
| core | core | `nanomcp` gateway (1 compose service, 1 binary) | The gateway IS the product; its core value (proxy MCP clients to upstream MCP servers) is exercised end-to-end by one service. Overreach guard: single-part product stays single-part. |

**Mandatory infra — attempted and REFUSED (recorded per §20, never silently dropped):**

| item | verdict | reason |
|------|---------|--------|
| database (postgresql + pgrust, §8) | REFUSED | Product mandates none: state is file-based (`servers.json`, `gateway-tokens.json`, credential files 0600/0700 under `userdata/`); no DB code in `src/`. |
| OIDC (kanidm, §10) | REFUSED | Product mandates none as external infra: the gateway itself implements the OAuth 2.1 authorization server for inbound auth (`/oauth/*`); no external IdP integration found in source. |
| S3 (s3-storage, §11) | REFUSED | No object-storage code or config surface in source; file-based persistence only. |

**Links requiring `step-ca-trust` (§14):** none. Single group, no inter-group
http-by-domain calls. Upstream MCP servers are third-party resources, not groups
of this product. (Publishing the gateway itself behind the SSL stack is an
environment-overlay concern, not a cross-group link.)

**Base stage plan:**

- **Stage A (core)** — scaffold `app/` with the source-built `nanomcp` service
  (`${IMAGE:-localhost/nanomcp:<tag>}` per §19), named volume `nanomcp-userdata`
  over the `userdata/` path, network `nanomcp-network`, `forwarding`
  environment. Run the functional gate scenario below.
- No outer stages: no other groups exist.

**Functional gate scenario (Stage A browser/HTTP gate):**

1. Container starts the gateway with a virtual display: the headless-boot
   hypothesis (§ Open hypotheses) was **REFUTED** at the gate — nanoui's GUI
   startup unconditionally brings GTK up (`startGtk` → `gtk.load()` dlopens
   `libgtk-4.so.1`/`libgtk-3.so.0`; `gtk_init_check` requires a display), and
   its headless modes (`--nanoui-snapshot`/`--nanoui-generate`) make `run()`
   return immediately, so they cannot host the gateway. Fix shipped in the
   image: runtime packages `xvfb libgtk-3-0`, entrypoint exports
   `NANOUI_GTK=3` + `DISPLAY=:99` and starts `Xvfb :99` before the gateway.
2. `GET /healthz` returns `{"status":"ok","tools":0}`.
3. Bring up the gate-enabling `stub-mcp-upstream` extension (a minimal upstream
   MCP server) and register it through the persisted `servers.json` inside the
   userdata volume (no GUI in the container).
   **DEVIATION (recorded 2026-11-05):** the stub is a **streamable-HTTP**
   upstream (`http://stub-mcp-upstream:8080/mcp`, python:3-alpine inline
   `command`), not the stdio-helper-container shape originally sketched here.
   Reasons: the gateway's stdio transport spawns child processes *inside the
   gateway container* (PATH/HOME inherited), so a separate helper container
   cannot be the stdio child without extra plumbing; a streamable-HTTP stub
   exercises the same gateway path (register → aggregate → proxy tool call)
   with a plain HTTP upstream. stdio remains available via the
   `mcp-gateway-stdio` extension idea for real stdio upstreams.
4. `GET /healthz` reports `tools: 1`; an MCP client connects to `/mcp`
   (inbound `token` mode, bearer = the seeded inbound token from
   `userdata/inbound-token.json`) and successfully calls the stub's tool
   through the gateway.

## Extension catalog

| extension | type | status | dependencies |
|-----------|------|--------|--------------|
| `stub-mcp-upstream` | product (gate companion) | `required-for-gate` — implement BEFORE Stage A gate | none; minimal **streamable-HTTP** upstream MCP server (python:3-alpine, inline `command`) the gate proxies through — see the DEVIATION note in the gate scenario (stdio-helper shape replaced) |
| `mcp-gateway-stdio` | product | `available` | container-side stdio upstream helpers (`sleep`-based child-process image shape); needed by any stdio upstream beyond the stub |
| `publish-gateway-url-discovery` | product | `implemented` | `/.well-known/oauth-protected-resource` discovery for a host-published gateway URL: `NANOMCP_PUBLIC_PORT` seeds `servers.json` `.port` (the only config surface the binary reads — metadata base is hardcoded `http://127.0.0.1:<port>`, no env var, no Host header) so the metadata advertises the published URL when `NANOMCP_PORT` == `NANOMCP_PUBLIC_PORT`. Upstream limitation: hostname-based SSL publishing is impossible (base is always `http://127.0.0.1:<port>`); `NANOMCP_PUBLIC_PORT` must differ from the relay listen port 7332. Inert when unset |

Statuses: `available` = recorded, NOT implemented without explicit user
confirmation at the extension-menu checkpoint (§20); `required-for-gate` =
implemented before the core gate, no confirmation needed.

## Open hypotheses

| hypothesis | confidence | what the functional gate must observe |
|------------|------------|----------------------------------------|
| ~~The gateway binary runs headless in a container: serving autostarts (`gw.startAsync`) without any window/display; nanoui init can be skipped or hidden via `--nanoui-hidden`-semantics flags~~ **REFUTED (2026-11-05 gate)** | medium | nanoui GUI startup unconditionally brings GTK up (`startGtk` → `gtk.load()` dlopens libgtk-4/libgtk-3; `gtk_init_check` needs a display); headless modes (`--nanoui-snapshot`/`--nanoui-generate`) make `run()` return immediately → cannot host the gateway. Resolution: Xvfb virtual display + GTK3 runtime in the image (see gate step 1) |
| stdio upstream transport works in-container: child processes inherit PATH/HOME (`gw.environ = proc.environ_map`), so a `sleep`-based helper image or in-image binary can serve as an upstream | medium | stub upstream registers and its tool call succeeds through `/mcp` |
| routes/upstream config can be pre-seeded by writing `servers.json` in the `nanomcp-data` volume (no GUI interaction) | medium | gateway adopts the pre-seeded upstream (healthz `tools: 1`) without GUI edits |
