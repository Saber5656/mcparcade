# Title

MCP record tools: `list_sessions`, `get_record`, `get_replay`

## Summary

Register the three identity/history tools per DESIGN.md §5.6–§5.8: session listing/resume support, the profile/ratings report, and paginated replay retrieval.

## Context

These tools are how the agent "remembers" — resuming interrupted games, reporting its rating, and reviewing past matches. They are read-only facades over the stores (issues 06/07).

## Scope

- `list_sessions {status?, gameId?}` → `{sessions: [{sessionId, gameId, status, turn, updatedAt, summary}], spectatorUrl}` (§5.6; ≤ 50 rows, newest first, default `active`).
- `get_record {gameId?}` → `{games: {<gameId>: <profile entry per §10.1, verbatim>}, anchorsNote: string}`; with `gameId`, the `games` map contains only that entry (no cross-game totals in v1).
- `get_replay {replayId, from?, to?}` → `{meta: {replayId, sessionId, gameId, gameVersion, seed, options, players, createdAt, endedAt}, outcome, total, from, to, entries: [{t, playerId, action, summary}]}` where each entry joins the action-log record with its same-turn event summary (§5.8; ≤ 500 entries/call).

## Detailed Requirements

1. `list_sessions` summary rows: `{sessionId, gameId, status, turn, updatedAt, summary}` where `summary` is the last public event's summary or the outcome summary. Never include state snapshots.
2. `get_record` computes nothing new — it projects `profile.json` (issue 07) plus rating anchors for context; response documents `rating` as "vs built-in difficulty anchors" (string constant).
3. `get_replay` pagination: `from`(default 0)/`to`(exclusive, default from+500, cap 500 span); out-of-range clamps; response includes `total` so the agent can iterate. Unknown replay id → `E_REPLAY_NOT_FOUND` (already in the §7.5 taxonomy); a quarantined replay file → `E_CORRUPT_SESSION`.
4. Security (T1/§18.4): all three tools use the issue-10 registrar (zod input schemas, 16 KiB request cap, response invariants); `replayId` validated with `isValidId("replay", …)` at the facade before any store call; all file access goes through issue-05 `pathFor`.
5. Empty states are first-class: no profile yet → default zeros; no sessions → empty array + text "no active sessions — call list_games to see the catalog, then start_game".
6. Text blocks: `get_record` renders a compact per-game table (monospace) — this is the "show my record" chat artifact.

## Acceptance Criteria

- [ ] After two finished stub-game sessions (via runtime), `list_sessions {status:"finished"}` returns both, newest first; `active` default hides them.
- [ ] `get_record` shows plays/rating matching the profile file; `gameId` filter works; empty profile returns zeros without error.
- [ ] `get_replay` slices: `from=0,to=2` returns 2 entries + `total`; `from≥total` returns an empty slice, not an error; unknown id → `error.code == "E_REPLAY_NOT_FOUND"`; malformed id (traversal string) rejected at the facade with the same code and zero store access (spy).
- [ ] All three tools are read-only: storage write spy asserts zero writes.

## Validation

- `test/integration/record-tools.test.ts` (in-memory client; refactor onto harness in 14).

## Dependencies

- 06, 07, 09 (tests finalize sessions through the runtime), 10.

## Non-goals

- Replay frame recompute (09/22), spectator endpoints (19), rating math (08).

## Design References

- DESIGN.md §5.6–§5.8, §7.5, §9.5, §10, §18.2 (T1), §18.4
