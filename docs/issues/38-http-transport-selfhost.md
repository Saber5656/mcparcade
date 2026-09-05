# Title

Streamable HTTP transport: single-user self-host mode with mandatory bearer token

## Summary

Implement `src/server/http.ts` and activate the `--http <port>` CLI family per DESIGN.md §17: the SDK's Streamable HTTP transport behind mandatory static-token auth, loopback-default binding, Host/Origin discipline, and a token-bucket rate limit.

## Context

This unlocks the "playable from AI chat" promise for claude.ai custom connectors and remote MCP clients (ADR-003) while staying a single-user personal deployment — not a hosted service.

## Scope

- `mcparcade serve --http 8765 [--bind 127.0.0.1] [--allow-host <host>]...` — MCP endpoint at `/mcp` via `StreamableHTTPServerTransport`, sharing the same `McpServer`/runtime as stdio (stdio remains active only when NOT in http mode; `--http` runs http-only — one process, one transport, keeps session semantics simple).
- Auth middleware: `Authorization: Bearer <MCPARCADE_HTTP_TOKEN>` required on every request incl. GET/SSE; env var mandatory, min 16 chars, server refuses to start otherwise with a setup hint (user creates the secret manually — never generated/persisted by us).
- Bind rules: default `127.0.0.1`; `--bind 0.0.0.0` (or any non-loopback) prints a multi-line stderr warning (TLS/proxy guidance); Host allowlist = loopback set + `--allow-host` entries (for reverse-proxy names).
- Rate limit (§17): token bucket 20 req/s, burst 40, keyed by (source IP, `mcp-session-id`); 429 with `Retry-After: 1`.
- Session management per SDK semantics (`mcp-session-id` header): support multiple concurrent MCP sessions from the same user; all share the one profile/data dir (documented).

## Detailed Requirements

1. Constant-time token compare; 401 JSON without `WWW-Authenticate` echoing anything sensitive; log auth failures at warn with source IP, never the presented token (T10).
2. Origin: when bound to loopback, apply §11.3 rules; when `--allow-host` is set, those hosts join both Host and Origin allowlists (scheme-agnostic host match; the proxy owns TLS).
3. The spectator stays on its own loopback port with its own token even in http mode (unchanged; remote spectator = v2).
4. Graceful shutdown drains SSE/streamable connections (5 s cap).
5. Manual verification (resolves U7; **maintainer-executed**, environment-dependent by nature): author `docs/research/selfhost-claudeai-runbook.md` first (prerequisites: a host reachable over public HTTPS via the maintainer's own TLS reverse proxy or tunnel — e.g. Tailscale Funnel or equivalent; `MCPARCADE_HTTP_TOKEN` set; `--allow-host <public-hostname>`); then the maintainer follows it with a real claude.ai custom connector and a second generic remote MCP client. Success = the connector completes `initialize` + `tools/list` and plays one maze turn. "Blocked" = the connector cannot complete `initialize` due to auth-scheme requirements (e.g. OAuth-only) — record the exact error, keep bearer support, file the v2 OAuth follow-up (do not implement OAuth here).
6. `--http` + `--no-spectator` + `--data-dir` compose; `serve` without `--http` behaves exactly as before (regression tests).

## Acceptance Criteria

- [ ] In-process HTTP integration test: full MCP handshake + `list_games` + a maze `start/act` round-trip over Streamable HTTP with the token; 401 without/with-wrong token; 429 after burst exhaustion (fake timer).
- [ ] Startup refusal matrix: no env token / 15-char token / non-loopback bind warning (stderr snapshot).
- [ ] Host/Origin matrix: loopback Host/Origin accepted; `Host: evil.example` → 403; `Origin: https://evil.example` → 403; with `--allow-host proxy.example`: `Host: proxy.example` accepted, other foreign hosts still 403.
- [ ] Two concurrent MCP sessions play two different games without cross-talk (session-id isolation at the transport layer; same profile by design).
- [ ] stdio mode untouched (issue 13's stdio tests still green).
- [ ] Runbook committed; manual claude.ai/remote-client verification performed and results recorded in the PR (or U7 finding documented if blocked).

## Validation

- `test/integration/http-transport.test.ts`; manual runbook execution logged in the PR.

## Dependencies

- 13.

## Non-goals

- OAuth/multi-user/tenancy (v2), TLS termination (user's proxy), remote spectator, rate-limit persistence.

## Design References

- DESIGN.md §17, §11.3, §18.2 (T10, T11), §21 (U7); ADR-003
