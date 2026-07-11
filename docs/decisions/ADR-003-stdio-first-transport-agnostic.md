# ADR-003: stdio-first, transport-agnostic core, single-user HTTP self-host in v1

Status: accepted (user decision, 2026-07-10)

## Context

Target clients split into local (Claude Code, Claude Desktop, Codex CLI, Cursor — stdio) and remote (claude.ai custom connectors — Streamable HTTP). A hosted multi-tenant service would require accounts, isolation, abuse handling, and infra — a different product.

## Decision

- v1 default: **stdio** (`npx mcparcade`).
- The core (runtime, games, storage, spectator) is **transport-agnostic**; transports are thin wiring (DESIGN.md §4.2).
- v1 also ships `--http <port>`: Streamable HTTP for **single-user self-hosting**, mandatory static bearer token via `MCPARCADE_HTTP_TOKEN`, loopback bind by default (§17).
- Official hosted service: v2, separate design round (OAuth, tenancy).

## Consequences

- Local users get the fastest path; remote chat users have a documented self-host path.
- Single-user HTTP means all authenticated clients share one profile — documented, acceptable for personal hosting.
- Cost: transport abstraction discipline (no transport types below `server/`), enforced by review.

## Alternatives rejected

- stdio-only: blocks the "playable from AI chat" promise for claude.ai users.
- Hosted-first: v1 scope explosion (auth, tenancy, abuse, ops) before the games exist.
