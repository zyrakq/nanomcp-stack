# UPSTREAM

## app/ — https://github.com/sternelee/nanomcp

| date | upstream commit | image tag / digest | note |
|------|-----------------|--------------------|------|
| 2026-11-05 | 90cffaaec6ea17f55f7cb8c13871c3f8f03a88f0 | `localhost/nanomcp:0.1.0` | source-built per conventions §19 (`app/images/nanomcp/Dockerfile`, Zig 0.17.1, nanoui pinned v0.12.0 in `build.zig.zon`); no published Docker image exists (Docker Hub `sternelee/nanomcp` 404) |
| 2026-11-05 | 90cffaaec6ea17f55f7cb8c13871c3f8f03a88f0 | `localhost/nanomcp:0.1.0` / `sha256:bea3b0570269f57380f48a1d8bdd0bcb176918b2a1026152215ac1520423258a` | rebuilt after gate: entrypoint seeds the inbound token at the flat `userdata/inbound-token.json` path (upstream `credentials.zig` `pathOf`, not a `credentials/` subdir) and runs the gateway behind Xvfb + GTK3 (`NANOUI_GTK=3`, `DISPLAY=:99`) — headless boot refuted, see DEPLOYMENT.md |
| 2026-11-05 | 90cffaaec6ea17f55f7cb8c13871c3f8f03a88f0 | `localhost/nanomcp:0.1.0` sha256:41d72d11fa023be5e65318727db0dedfeb4f0a6da0e141f0f804409a8a3e0e1f | rebuilt after entrypoint fix: inbound token seeded at flat `<userdata>/inbound-token.json` per upstream `credentials.zig pathOf` (was `userdata/credentials/`); Stage A gate passed on this image |
