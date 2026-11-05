# syntax=docker/dockerfile:1
# nanomcp — source-built image (Zig 0.17.1).
# Upstream: https://github.com/sternelee/nanomcp @ 90cffaaec6ea17f55f7cb8c13871c3f8f03a88f0
# Self-contained: clones upstream at the pinned commit; build with an empty context:
#   docker build -t localhost/nanomcp:0.1.0 app/images/nanomcp
#
# The gateway binary binds 127.0.0.1:7331 (hardcoded upstream, no env var), so
# the entrypoint runs a socat relay on 0.0.0.0:7332 -> 127.0.0.1:7331 and
# compose publishes only the relay port.

FROM debian:stable-slim AS build

ARG NANOMCP_REPO=https://github.com/sternelee/nanomcp.git
ARG NANOMCP_COMMIT=90cffaaec6ea17f55f7cb8c13871c3f8f03a88f0
ARG ZIG_VERSION=0.17.1
ARG ZIG_SHA256=3581863e3cbcfd0fce0efc682ce7e9e93da96668a977791d356a26d2a7e9b17a

RUN apt-get update \
    && apt-get install -y --no-install-recommends git curl ca-certificates xz-utils \
    && rm -rf /var/lib/apt/lists/*

# Zig toolchain: official tarball, checksum-verified.
RUN curl -fsSL "https://ziglang.org/download/${ZIG_VERSION}/zig-x86_64-linux-${ZIG_VERSION}.tar.xz" -o /tmp/zig.tar.xz \
    && echo "${ZIG_SHA256}  /tmp/zig.tar.xz" | sha256sum -c - \
    && mkdir -p /opt/zig \
    && tar -xJf /tmp/zig.tar.xz -C /opt/zig --strip-components=1 \
    && rm /tmp/zig.tar.xz
ENV PATH="/opt/zig:${PATH}"

WORKDIR /build
RUN git clone "${NANOMCP_REPO}" src \
    && cd src \
    && git checkout "${NANOMCP_COMMIT}"

# ReleaseSafe build; zig fetches nanoui itself (pinned v0.12.0 in
# build.zig.zon, fetched from git+https).
RUN cd src && zig build -Doptimize=ReleaseSafe

FROM debian:stable-slim

# The binary embeds a nanoui GUI whose startup unconditionally brings GTK up
# (nanoui's startGtk -> gtk.load/gtk_init_check), so the runtime needs the GTK
# libraries AND a display even though no window is ever shown. Xvfb provides a
# virtual X display; NANOUI_GTK=3 (set in the entrypoint) forces the GTK3
# backend, which renders on cairo/CPU — no GPU/OpenGL needed in a container.
RUN apt-get update \
    && apt-get install -y --no-install-recommends socat jq curl ca-certificates \
        xvfb libgtk-3-0 \
    && rm -rf /var/lib/apt/lists/* \
    && mkdir -p /tmp/.X11-unix && chmod 1777 /tmp/.X11-unix \
    && useradd --create-home --shell /usr/sbin/nologin nanomcp

COPY --from=build /build/src/zig-out/bin/nanomcp /usr/local/bin/nanomcp

# Pre-create the userdata mount point owned by the runtime user, so the named
# volume inherits the right ownership on first use.
RUN mkdir -p /home/nanomcp/.local/share/nanomcp/userdata \
    && chown -R nanomcp:nanomcp /home/nanomcp

# mcp-gateway-stdio extension helper: a minimal MCP stdio upstream (echo
# tool) speaking line-delimited JSON-RPC on its standard streams — exactly
# what the gateway's stdio transport (transport_stdio.zig) spawns and pipes.
# Doubles as a template for real stdio servers; needs only jq (installed
# above). Written inline (empty build context — nothing is COPYed).
RUN <<'EOF'
cat > /usr/local/bin/mcp-stdio-echo <<'SH'
#!/bin/sh
# Minimal MCP stdio upstream: one JSON-RPC message per line, echo tool.
while IFS= read -r line; do
  [ -n "${line}" ] || continue
  resp="$(printf '%s' "${line}" | jq -c '
    .method as $m
    | if $m == null or ($m | startswith("notifications/")) then empty
      elif $m == "initialize" then
        {jsonrpc: "2.0", id: .id,
         result: {protocolVersion: "2025-06-18", capabilities: {tools: {}},
                  serverInfo: {name: "mcp-stdio-echo", version: "0.1.0"}}}
      elif $m == "tools/list" then
        {jsonrpc: "2.0", id: .id,
         result: {tools: [{name: "echo",
                           description: "Echo back the input text",
                           inputSchema: {type: "object",
                             properties: {text: {type: "string"}},
                             required: ["text"]}}]}}
      elif $m == "tools/call" then
        {jsonrpc: "2.0", id: .id,
         result: {content: [{type: "text",
                  text: ("echo: " + (.params.arguments.text // ""))}]}}
      else
        {jsonrpc: "2.0", id: .id,
         error: {code: -32601, message: ("method not found: " + $m)}}
      end')"
  [ -n "${resp}" ] && printf '%s\n' "${resp}"
done
SH
chmod 0755 /usr/local/bin/mcp-stdio-echo
EOF

# Entrypoint written inline (empty build context — nothing is COPYed).
RUN <<'EOF'
cat > /usr/local/bin/nanomcp-entrypoint <<'SH'
#!/bin/sh
set -eu

DATA_DIR="${HOME}/.local/share/nanomcp/userdata"
TOKEN_FILE="${DATA_DIR}/inbound-token.json"
SERVERS_FILE="${DATA_DIR}/servers.json"
TOKEN="${NANOMCP_SEED_INBOUND_TOKEN:-}"
STUB_URL="${NANOMCP_SEED_STUB_URL:-}"

mkdir -p "${DATA_DIR}"

# Inbound token: the credential store keeps it as a flat file next to
# servers.json (credentials.zig pathOf: <userdata>/inbound-token.json, 0600).
# Seeded only when the env var is set and differs from what is on disk, so a
# token set through the GUI is never clobbered for no reason.
if [ -n "${TOKEN}" ]; then
  current=""
  [ -f "${TOKEN_FILE}" ] && current="$(cat "${TOKEN_FILE}" 2>/dev/null || true)"
  if [ "${current}" != "${TOKEN}" ]; then
    printf '%s' "${TOKEN}" > "${TOKEN_FILE}"
    chmod 600 "${TOKEN_FILE}"
  fi
fi

# Stub upstream: create-or-upsert one servers.json entry. Only runs when
# NANOMCP_SEED_STUB_URL is set; every other entry (GUI-managed) is left
# untouched. (The inbound "token" flip is a standalone step below, so it
# applies no matter which seeding block created the file.)
if [ -n "${STUB_URL}" ]; then
  if [ ! -f "${SERVERS_FILE}" ]; then
    printf '{"port":7331,"servers":[],"inbound":"off"}\n' > "${SERVERS_FILE}"
  fi
  jq --arg url "${STUB_URL}" '
    .servers = ((.servers // []) | map(select(.id != "stub-mcp-upstream"))
      + [{"id":"stub-mcp-upstream","name":"Stub MCP","alias":"stub",
          "transport":"http","enabled":true,"url":$url}])' \
    "${SERVERS_FILE}" > "${SERVERS_FILE}.tmp" \
    && mv "${SERVERS_FILE}.tmp" "${SERVERS_FILE}"
fi

# Stdio helper upstream (mcp-gateway-stdio extension): create-or-upsert one
# servers.json entry pointing at the in-image helper script. Only runs when
# NANOMCP_STDIO_HELPER=1; every other entry (GUI-managed) is left untouched.
# The gateway spawns stdio children inside its own container
# (transport_stdio.zig), so the helper must live in the same image.
if [ "${NANOMCP_STDIO_HELPER:-}" = "1" ]; then
  if [ ! -f "${SERVERS_FILE}" ]; then
    printf '{"port":7331,"servers":[],"inbound":"off"}\n' > "${SERVERS_FILE}"
  fi
  jq '
    .servers = ((.servers // []) | map(select(.id != "stdio-helper"))
      + [{"id":"stdio-helper","name":"Stdio Helper","alias":"stdio-helper",
          "transport":"stdio","enabled":true,
          "command":"/usr/local/bin/mcp-stdio-echo"}])' \
    "${SERVERS_FILE}" > "${SERVERS_FILE}.tmp" \
    && mv "${SERVERS_FILE}.tmp" "${SERVERS_FILE}"
fi

# mcp-gateway-stdio extension: seed one stdio upstream. NANOMCP_SEED_STDIO_JSON
# holds a complete servers.json server object (transport "stdio"; command is
# required and resolved inside this container via the child's PATH — the image
# ships the mcp-stdio-echo helper above). Empty = no stdio seeding; every
# other entry (GUI-managed) is left untouched. Upserted by id.
STDIO_JSON="${NANOMCP_SEED_STDIO_JSON:-}"
if [ -n "${STDIO_JSON}" ]; then
  if ! printf '%s' "${STDIO_JSON}" | jq -e 'type == "object"
      and (.id // "" | length > 0)
      and (.name // "" | length > 0)
      and (.alias // "" | length > 0)
      and (.command // "" | length > 0)' > /dev/null; then
    echo "nanomcp-entrypoint: invalid NANOMCP_SEED_STDIO_JSON" >&2
    exit 1
  fi
  if [ ! -f "${SERVERS_FILE}" ]; then
    printf '{"port":7331,"servers":[],"inbound":"off"}\n' > "${SERVERS_FILE}"
  fi
  jq --argjson srv "${STDIO_JSON}" '
    .servers = ((.servers // []) | map(select(.id != $srv.id)) + [$srv])' \
    "${SERVERS_FILE}" > "${SERVERS_FILE}.tmp" \
    && mv "${SERVERS_FILE}.tmp" "${SERVERS_FILE}"
fi

# Inbound mode: flip servers.json .inbound to "token" when a token was
# seeded (unless the file already uses oauth — GUI-managed). Standalone step
# after all seeding blocks, so every path that creates servers.json fresh
# (stub, stdio helper, generic JSON) ends in the right mode even when no
# other block runs.
if [ -f "${SERVERS_FILE}" ] && [ -n "${TOKEN}" ]; then
  jq 'if (.inbound // "off") == "oauth" then . else .inbound = "token" end' \
    "${SERVERS_FILE}" > "${SERVERS_FILE}.tmp" \
    && mv "${SERVERS_FILE}.tmp" "${SERVERS_FILE}"
fi

# Public URL discovery (publish-gateway-url-discovery extension): the binary
# derives every /.well-known/oauth-* metadata base from the port configured in
# servers.json (gateway_views.zig baseURL: "http://127.0.0.1:<port>" — no env
# var, no Host header). NANOMCP_PUBLIC_PORT seeds that port so the metadata
# advertises the URL published on the host. Empty = leave the configured port
# alone (default 7331). Only touched when set and differing, so a port saved
# through the GUI is never clobbered for no reason.
PUBLIC_PORT="${NANOMCP_PUBLIC_PORT:-}"
case "${PUBLIC_PORT}" in
  ''|*[!0-9]*) PUBLIC_PORT="" ;;
esac
if [ -n "${PUBLIC_PORT}" ]; then
  if [ ! -f "${SERVERS_FILE}" ]; then
    printf '{"port":%s,"servers":[],"inbound":"off"}\n' "${PUBLIC_PORT}" > "${SERVERS_FILE}"
  elif [ "$(jq -r '.port // 7331' "${SERVERS_FILE}")" != "${PUBLIC_PORT}" ]; then
    jq --argjson p "${PUBLIC_PORT}" '.port = $p' "${SERVERS_FILE}" > "${SERVERS_FILE}.tmp" \
      && mv "${SERVERS_FILE}.tmp" "${SERVERS_FILE}"
  fi
fi
# The gateway serves on the port configured in servers.json (default 7331);
# the relay below must forward to exactly that port.
GW_PORT="$(jq -r '.port // 7331' "${SERVERS_FILE}" 2>/dev/null || echo 7331)"

# The binary embeds a nanoui GUI: its startup brings GTK up unconditionally
# (nanoui startGtk -> gtk.load/gtk_init_check), which needs a display even
# though no window is ever shown here. Serve it a virtual X display. NANOUI_GTK=3
# forces the GTK3 backend (cairo/CPU rendering — no GPU in a container).
export NANOUI_GTK=3
export DISPLAY=:99
Xvfb :99 -screen 0 1280x800x24 -ac -nolisten tcp &
i=0
while [ ! -S /tmp/.X11-unix/X99 ] && [ "${i}" -lt 50 ]; do
  sleep 0.1
  i=$((i + 1))
done

# The gateway binds 127.0.0.1:<configured port> (upstream hardcodes the
# address, servers.json holds the port); expose it through a socat relay on
# all interfaces so compose can publish one port.
/usr/local/bin/nanomcp &
exec socat TCP-LISTEN:7332,fork,reuseaddr TCP:127.0.0.1:"${GW_PORT}"
SH
chmod 0755 /usr/local/bin/nanomcp-entrypoint
EOF

ENV HOME=/home/nanomcp
USER nanomcp

# /healthz requires the inbound bearer in token mode, so the healthcheck only
# verifies the gateway answers HTTP at all (200 in off mode, 401 in token
# mode — curl without --fail accepts both; connection refused fails it).
# (dash's /bin/sh has no /dev/tcp, so a shell TCP probe would never work.)
HEALTHCHECK --interval=10s --timeout=5s --retries=12 --start-period=30s \
    CMD curl -s -o /dev/null -m 4 http://127.0.0.1:7331/healthz || exit 1

ENTRYPOINT ["/usr/local/bin/nanomcp-entrypoint"]
