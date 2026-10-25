# REA — agentic integration deep dive

What follows is the contract REA exposes to an MCP client, independent of how
it's deployed in this lab. Useful for registering REA against a tool that
isn't wired through the `wn-mcp-01` gateway (a laptop-local Claude Code
session, Codex, Cursor, Gemini CLI), or for understanding what the gateway is
actually proxying.

Verified against REA's `README.md` and `docs/installation.md`
(`morluto/rea`, 2026-10-25).

## What REA is

"One MCP for reverse engineering across binaries, applications, and runtime
behavior." Covers native binaries, JavaScript/Electron apps, .NET assemblies,
Android APKs, Flutter, PyInstaller, Apple bundles, Unity IL2CPP, Godot PCK,
websites, firmware, HAR/SAZ/PCAP captures, SQLite, GGUF/safetensors, GPU code
objects, and recorded crashes. Everything runs locally; results include
evidence and stated limitations.

**Requirements:** Node.js `22.x` (≥22.23.4), `24.x` (≥24.22.1), or `26.12.1+`,
plus npm.

## Manual MCP registration (JSON)

The shape any MCP client config needs, pinned:

```json
{
  "mcpServers": {
    "rea": {
      "command": "npx",
      "args": ["-y", "rea-agents@6.7.0", "mcp"]
    }
  }
}
```

With Bun: `bunx rea-agents@<version> mcp` in place of `npx -y`. On Windows,
setup wraps the command in `cmd.exe /d /c`.

## Guided setup (per-agent)

```bash
npx rea-agents setup
```

Registers REA's MCP server and installs workflow instructions for supported
agents (Claude Code, Codex, Cursor, Gemini CLI, Grok Build, others). Shows
proposed changes for approval, backs up existing config. **Restart the agent
afterward.**

### Claude Code specifically

- Default `setup` writes `mcpServers.rea` to `~/.claude.json` and installs the
  skill to `~/.claude/skills` — applies to every project.
- Project-only registration (no skill install, `rea update` won't refresh it):
  ```bash
  claude mcp add --scope local rea -- npx -y rea-agents@6.7.0 mcp
  ```
  Windows: `claude mcp add --scope local rea -- cmd.exe /d /c npx -y rea-agents@6.7.0 mcp`
  Avoid `--scope project` — writes a committable `.mcp.json`.
- Separate profile:
  ```bash
  CLAUDE_CONFIG_DIR=~/.claude-rea npx rea-agents setup --client claude_code
  CLAUDE_CONFIG_DIR=~/.claude-rea claude
  ```

### Codex

`--client codex` pins the registration to the installed version, adds
`startup_timeout_sec = 30` and a derived `tool_timeout_sec`.

### Cursor / Gemini CLI

`--client cursor` / `--client gemini_cli`. Config paths not covered here — see
REA's own `client-configuration.md`.

**All clients:** restart after setup. Quit Claude Desktop fully before
approving setup — a running app can overwrite its own registration mid-write.

## CLI usage (outside any agent)

```bash
# One-off JS/Electron/ASAR analysis, no MCP registration needed
npx -y rea-agents@latest analyze-javascript-application /absolute/path/to/app --json

# Global install
npm install --global rea-agents
rea --help

# Non-interactive, scoped, dry-run first
rea setup --client codex --client cursor --skill=false --dry-run

# Unattended, including Hopper (requires explicit approval flags — we don't use this)
rea setup --yes --all-detected --install-hopper --json

# Skill-only install, no MCP registration
npx skills add morluto/rea --skill reverse-engineer-anything

# Diagnostics / maintenance
rea doctor --json
rea doctor --client codex --json
rea update
rea uninstall
rea uninstall --purge-data
```

## Environment variables (client-side)

**Profiles and paths**
- `CLAUDE_CONFIG_DIR` — separate Claude Code profile
- `XDG_CACHE_HOME` — base for Hopper release-archive cache on Linux
- `REA_KUNA_CACHE_DIR` — Kuna cache override

**MCP and logging**
- `REA_LOG_LEVEL` — invalid values fail every scope
- `REA_MCP_INPUT_SCHEMA_PROFILE` / `REA_MCP_OUTPUT_SCHEMA_PROFILE` /
  `REA_MCP_MAX_RESPONSE_BYTES` — retained in the server entry by setup
- `REA_ANALYSIS_PROVIDER` — `ghidra` or `hopper` when a target supports both
  (our deployment only installs Ghidra, so this is hardcoded)

**Hopper** (not installed in our deployment — reference only)
- `HOPPER_LAUNCHER_PATH`
- `REA_HOPPER_STARTUP_TIMEOUT_MS` — 1 to 2,147,483,647 ms (default 120,000)

**Ghidra**
- `GHIDRA_INSTALL_DIR` — absolute path to a Ghidra 12.1.x installation
- `JAVA_HOME` — 64-bit JDK within Ghidra's declared Java range (21+ for 12.1.4)
- `REA_GHIDRA_STARTUP_TIMEOUT_MS` — default 330,000
- `REA_GHIDRA_DECOMPILE_TIMEOUT_SECONDS` — unbounded if unset
- `REA_GHIDRA_LANGUAGE_ID` / `REA_GHIDRA_COMPILER_SPEC_ID` — import language
  overrides; compiler spec requires a language
- `REA_GHIDRA_SEED_FILE` — TSV of `address<TAB>kind[<TAB>name]`
- `REA_GHIDRA_DISABLED_ANALYZERS` — comma-separated exact analyzer names
- `GHIDRA_HEADLESS_MAXMEM` (then `GHIDRA_MAXMEM`) — JVM heap, default `2G`

**Other**
- `REA_SWIFTC_COMPILE_TIMEOUT_MS` — ownership-helper compile deadline on
  macOS, default 120,000 ms

## Workflow shape

The calling agent reaches REA over MCP to inspect a target and trace code.
REA returns findings with evidence; the agent uses them to explain behavior or
write and test an implementation against it. CLI commands follow the same
analysis workflows outside any agent loop.

## Documentation map (upstream)

- Installation and setup: `docs/installation.md`
- CLI and evidence: `docs/cli.md`
- MCP contracts and prompts: `docs/mcp-contracts.md`, `docs/mcp-prompts.md`
- Tool catalog: `docs/mcp-contracts.md#generated-catalog`
- Guides and showcases: https://rea.tools/
