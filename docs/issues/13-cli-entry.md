# Title

CLI entry: `serve` command, flags, env precedence, stdio transport wiring

## Summary

Implement `src/index.ts` + `src/server/stdio.ts` for real: `mcparcade` (default `serve`) boots storage, registry, runtime, MCP server, stdio transport, and (via a hook filled by issue 19) the spectator; `validate-scenario` is registered as a stub that issue 28 completes.

## Context

This is the executable users configure in their MCP clients (`npx -y mcparcade`). stdio protocol discipline (stdout is JSON-RPC only) is a hard requirement (research doc; DESIGN.md §7.6/§16).

## Scope

- Commander program: `serve` (default when no args), `validate-scenario <file> [--json]` (full flag surface parses and shows in `--help`; the stub prints "not implemented" to stderr and exits 2 regardless of flags, until issue 28), `--version`, `--help`.
- `serve` flags: `--data-dir <path>`, `--no-spectator`, `--spectator-port <n>`, `--http <port>`, `--bind <addr>`, `--allow-host <host>` (repeatable) — `--http`-family flags parse now but exit with "HTTP mode lands in a later release" until issue 38 (feature-gated, so flag names are stable).
- Env: `MCPARCADE_DATA_DIR`, `NO_COLOR` consumed now; `MCPARCADE_HTTP_TOKEN` documented in `--help` with a "(HTTP mode — see --http)" note but consumed only by issue 38. Precedence flag > env > default (§16).
- Graceful shutdown on SIGINT/SIGTERM: close transport, flush pending storage writes, exit 0.

## Detailed Requirements

1. Boot order: resolve config → init Storage (issue 05) → build registry (static imports of game modules; empty until wave 1) → Runtime → `createMcpServer` → connect `StdioServerTransport` → start spectator unless disabled (call the injected `startSpectator(config, deps)` seam; before issue 19 it's a no-op returning `{url: null}`).
2. In `serve`, install a guard that patches `console.log` to write to stderr with a warning (belt-and-braces for stdout discipline; §19.4's grep-guard is the primary defense).
3. All startup logs to stderr: version, data dir, spectator URL (when enabled), "ready on stdio". Every interpolated value in log lines — including the resolved data-dir path and error messages — passes through `sanitizeForLog` (issue 09; §7.6 control-char stripping), since flags/env are user-controlled input.
4. Subcommand output (e.g. future `validate-scenario`) may use stdout freely — only `serve` is stdio-sacred.
5. Exit codes: 0 clean, 1 unexpected error, 2 usage error. Startup failures (unwritable data dir) print a clear stderr message, exit 1.
6. `--help` output documents every flag and env var with defaults.

## Acceptance Criteria

- [ ] `printf '' | node dist/index.js serve` starts and exits cleanly on stdin EOF; nothing but JSON-RPC ever appears on stdout (test drives one `initialize` + `tools/list` via a child-process stdio client and asserts stdout parses as pure JSON-RPC frames).
- [ ] MCP `tools/list` over real stdio returns all 8 MCP tools of DESIGN.md §5 (independent of how many games are registered; a stub game is injected via the test-only registry hook only so that `list_games`' response is non-empty in the same test).
- [ ] Flag/env precedence table test for data dir (flag beats env beats default), using temp dirs.
- [ ] SIGINT during idle exits 0 within 2 s.
- [ ] `mcparcade --version` prints version to stdout, exits 0; unknown flag exits 2 with usage on stderr.

## Validation

- `test/integration/cli.test.ts` (child-process based, temp HOME/data dirs).

## Dependencies

- 10, 11, 12.

## Non-goals

- HTTP transport implementation (38), spectator implementation (19), scenario validation (28).

## Design References

- DESIGN.md §16, §7.6, §17 (flag names only), §19.4 (T14)
