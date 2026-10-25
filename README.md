# witty-rea

Documentation, deployment reference, and analysis workspace for **REA**
(Reverse Engineer Anything — https://github.com/morluto/rea) as run inside
the Wittycomp Lab.

**This repo does not host the running service.** The actual deployment —
`Dockerfile`, `compose.yaml` service block, `nginx.conf` routing — lives in
the Config Pillar, same as every other lab service:

> `wittycomp-lab/apps/wn-mcp-01/servers/rea/`

That stays the single source of truth so nothing here can silently drift from
what's actually running. This repo is the front door: what REA is, why it's
deployed this way, how to use it, and where findings from real analysis work
end up.

## Contents

- [`docs/deployment.md`](docs/deployment.md) — mirrors `wn-mcp-01/servers/rea/README.md`:
  architecture, version pins, env vars, build/update/troubleshooting.
- [`docs/agentic-integration.md`](docs/agentic-integration.md) — the deep-dive
  on REA's MCP contract: manual registration syntax, CLI commands, env vars,
  per-agent setup (Claude Code, Codex, Cursor, Gemini CLI).
- [`workspace/`](workspace/) — scratch space for real analysis sessions:
  imported-binary notes, findings write-ups, anything worth keeping past one
  conversation. Not auto-populated; an agent or bearboss drops things here
  deliberately.

## Quick reference

| | |
|---|---|
| Endpoint | `http://10.10.30.40/rea` (via `wn-mcp-01` gateway) |
| Container | `wn-mcp-rea` · `172.23.0.82:8000` on `mcp_internal` |
| Upstream | `morluto/rea`, npm `rea-agents@6.7.0` |
| Native-binary backend | Ghidra `12.1.4` (baked in; no Hopper/IDA — commercial, unlicensed here) |
| Canonical config | `wittycomp-lab/apps/wn-mcp-01/servers/rea/` |
| Added | 2026-10-25 |

## Why REA

Ad hoc, task-driven: inspecting a vendor binary, understanding an
undocumented API a third-party app talks to, tracing what an unknown
executable does, recovering behavior from software with no surviving source.
Any Congregation agent reaches it the same way it reaches any other MCP
server in the lab — through the `wn-mcp-01` gateway.
