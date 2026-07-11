# ADR-001: One MCP server with a uniform tool surface

Status: accepted (user decision, 2026-07-10) · Supersedes the README's literal "collection of MCP servers" phrasing

## Context

The README described "a collection of mini-game MCP servers". Two viable shapes:
(a) one npm package per game, each its own MCP server; (b) one server hosting all games. Per-game tools (e.g. `shogi_move`, `werewolf_vote`) would exceed 30 tools across six games, bloating every client's tool list and system prompt; multiple servers multiply config entries, releases, and version skew.

## Decision

Ship **one** server (`mcparcade`) with **eight generic tools** (`list_games`, `describe_game`, `start_game`, `observe`, `act`, `list_sessions`, `get_record`, `get_replay`). Games are in-repo plugins implementing `GameDefinition` (DESIGN.md §6.1). Game-specific behavior lives in per-game *action schemas* and observation payloads, not in per-game tools.

## Consequences

- One config entry, one release, one version. Tool count stays constant as games are added.
- Cross-game features (records, ratings, replays, spectator) are trivially shared — they are the product's differentiator.
- Cost: `act` payloads are game-discriminated unions; `describe_game` must teach the action format well, and rejection hints must be excellent (DESIGN.md §7.3).
- The README will be updated to "a mini-game arcade MCP server" (issue 41).

## Alternatives rejected

- Per-game servers: config sprawl, no shared identity/records, N× release overhead.
- Single server with per-game tools: 30+ tools pollute agent context; naming collisions; breaks the "add a game without changing the API" property.
