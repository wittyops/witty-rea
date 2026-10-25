# Deployment reference

> This mirrors `wittycomp-lab/apps/wn-mcp-01/servers/rea/README.md`, the
> canonical copy. If the two ever disagree, the one in `wittycomp-lab` is
> right — update this file to match, not the other way around.

## What it is

REA is an MCP server that gives an agent tools to inspect and explain software
it doesn't have source for — native binaries, JS/Electron apps, .NET
assemblies, Android APKs, Flutter/PyInstaller/Apple bundles, websites,
firmware images, packet captures, SQLite/GGUF/safetensors files, and more. It
doesn't reimplement analysis itself; it wraps existing tools (Ghidra, JADX,
Binwalk/Unblob, ...) and returns findings with evidence, so the calling agent
can explain behavior or write and test an implementation against it.

It runs entirely locally — nothing analyzed here leaves the container.

## Architecture — why this isn't the usual `wn-mcp-base` pattern

Most services under `wn-mcp-01` are `wn-mcp-base:latest` + a hand-written
`servers/<name>/server.mjs` that implements MCP tool logic against some
upstream REST API. REA doesn't need that: `rea-agents` is a published npm
package that already speaks MCP over stdio (`npx rea-agents mcp`). There's no
business logic to write — only a transport to wrap. This follows the
`mcp-playwright` pattern instead: a custom Docker image with `supergateway`
wrapping the upstream package's own command, built by hand via `build.sh`
(compose has no `build:` key for this service, by design — see Gotchas below).

```
agent → wn-mcp-01 (nginx /rea) → wn-mcp-rea (supergateway, --stateful)
                                       └─ npx rea-agents@6.7.0 mcp (stdio)
                                            └─ Ghidra 12.1.4 (native binary analysis)
```

## What's baked in

| Component | Version | Why pinned |
|---|---|---|
| Node.js | 22.x (NodeSource) | REA requires ≥22.23.4 / ≥24.22.1 / ≥26.12.1 |
| `rea-agents` | `6.7.0` | `@latest` is a moving target that can break a running service without warning |
| Ghidra | `12.1.4` | REA's docs declare support for "Ghidra 12.1.x" specifically, not 12.2 |
| JDK | Eclipse Temurin 21 | Ghidra 12.1.4's own `application.properties` declares `java.min=21` |
| `supergateway` | `4.0.0` | See Gotchas — an unpinned version already regressed once elsewhere in this stack |

**No Hopper, no IDA.** Both are supported REA analysis providers, but both are
commercial tools requiring their own separate license — REA grants none
itself, and this deployment doesn't install them.

## Environment variables in play

Set in the Dockerfile (nothing here is a secret, so none of this lives in `.env`):

- `GHIDRA_INSTALL_DIR=/opt/ghidra`
- `JAVA_HOME=/opt/java/openjdk`
- `REA_ANALYSIS_PROVIDER=ghidra` — the only installed provider
- `REA_MCP_INPUT_SCHEMA_PROFILE=full` — complete tool-schema guidance, since
  several different agents call this server, not one tuned client
- `REA_GHIDRA_STARTUP_TIMEOUT_MS=330000` — Ghidra's own default

Available if a real need shows up: `REA_GHIDRA_DECOMPILE_TIMEOUT_SECONDS`,
`REA_GHIDRA_SEED_FILE`, `REA_GHIDRA_DISABLED_ANALYZERS`,
`GHIDRA_HEADLESS_MAXMEM` (default `2G`), `REA_LOG_LEVEL`,
`REA_MCP_MAX_RESPONSE_BYTES`.

## Building and deploying

```bash
cd wittycomp-lab/apps/wn-mcp-01
./servers/rea/build.sh
docker compose -f compose.yaml up -d mcp-rea
docker exec wn-mcp-01 nginx -s reload
```

## Gotchas

- **`compose_build` (the `lab-docker` MCP tool) does nothing for this
  service.** `mcp-rea` has no `build:` key in `compose.yaml` — it's built by
  hand on purpose, same as `mcp-playwright`, because the image is a fixed
  artifact (Ghidra baked in) rather than something that should rebuild on
  every compose cycle. Always run `build.sh` directly. Found live 2026-10-25:
  `compose_build mcp-rea` reported "Build complete" and did nothing, then
  `compose_up` failed with `pull access denied for wn-mcp-rea` because the
  image had never actually been built.
- **supergateway must stay pinned, `--stateful`, with `--healthEndpoint
  /healthz`.** An unpinned `supergateway` already regressed the
  `mcp-bitwarden` healthcheck once: a Docker healthcheck POSTing a full MCP
  `initialize` call every 30s against a `--stateful` endpoint spawned a
  permanently-living child process per probe — 130 phantom processes over 8
  days of uptime. The fix is the `--healthEndpoint` flag (a plain GET that
  never touches session machinery); don't remove it to "simplify" a future
  edit of this Dockerfile.

## Updating

1. `npm view rea-agents version` for the current `rea-agents` release
2. Check Ghidra's latest **12.1.x** patch (not 12.2+ unless REA's docs now say
   they support it) via the GitHub releases API, and grab the new asset's
   published `digest` for the Dockerfile's `GHIDRA_SHA256`
3. Update the `ARG` defaults in `Dockerfile`
4. `./build.sh`, then `docker compose up -d --force-recreate mcp-rea`
5. Confirm healthy: `docker inspect wn-mcp-rea --format '{{.State.Health.Status}}'`

## Licensing

REA's own code is MIT. Ghidra is Apache-2.0 (NSA, public release). Analyzing
software you don't own or aren't authorized to examine is on the operator, not
REA or this deployment.
