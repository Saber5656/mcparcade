# Title

Session and replay stores: schemas, CRUD, finalization extract, pruning

## Summary

Implement `src/storage/sessions.ts` and `src/storage/replays.ts` on top of the JSON store: typed persistence for session files (DESIGN.md §9.2) and replay files (§9.5), including the finished→replay extraction and FIFO pruning.

## Context

Sessions are the runtime's unit of persistence (ADR-004: seed + action log is the source of truth; `stateSnapshot` is a resume cache). Replays feed `get_replay`, the spectator replay viewer, and the determinism suite.

## Scope

- zod schemas `zSessionFile` (v1) and `zReplayFile` (v1). `zSessionFile` fields (all required unless noted): `schemaVersion, sessionId, gameId, gameVersion, seed, options, createdAt, updatedAt, status, players, actionLog, eventLog, stateSnapshot, rngCursors, frozen (default false), outcome (nullable)` — exactly DESIGN.md §9.2. `zReplayFile` fields: `schemaVersion, replayId, sessionId, gameId, gameVersion, seed, options, players, createdAt, endedAt, actionLog, eventLog, outcome, finalState` (= the session's last `stateSnapshot`); `rngCursors`/`frozen` are **not** copied into replays.
- `SessionStore`: `create`, `load`, `save`, `list(filter)`, `countActive`, lazy 30-day abandon (§7.1).
- `ReplayStore`: `writeFromSession(session)`, `load`, `list`, pruning.
- Pruning: keep newest 200 finished/abandoned session files (order key `updatedAt`), 500 replays (order key `endedAt`); prune runs after each finalization write. Active sessions are never pruned.
- All file paths are obtained exclusively through `pathFor`/`listJsonFiles` from issue 05 — this module performs no path construction of its own (DESIGN.md §18.4; covered by issue 05's grep-guard).

## Detailed Requirements

1. Session file fields exactly as DESIGN.md §9.2 (ids, seed, options, timestamps, status, players, actionLog, eventLog, stateSnapshot, outcome). Timestamps ISO-8601 UTC.
2. `load` of an `active` session with `updatedAt` older than 30 days rewrites it as `abandoned` before returning — no background timers. The full stale outcome (normative): `{result:"abandoned", reason:"stale", score:null, par:null, ratingDelta:null, rated:false, summaryExtras:null, summary:"abandoned after 30 days of inactivity"}`.
3. `list(filter)` supports `status` and `gameId`, returns newest-first by `updatedAt`, caps at 50 summaries; summaries never include `stateSnapshot` (size).
4. Replay extraction copies: metadata, seed, options, players, full `actionLog` + `eventLog`, `outcome`, final `stateSnapshot`, `gameVersion`. Replay id: fresh `rep_` ULID; back-reference `sessionId` kept.
5. Pruning deletes oldest entries beyond caps (sessions by `updatedAt`, replays by `endedAt`); never deletes `active` sessions; logs each deletion at `info` level to stderr (§7.6 levels: info|warn|error).
6. Event/actionLog size enforcement helpers: `assertAppendable(session)` throws a typed signal when §7.4 caps (20 000 events / 2 MiB serialized) would be exceeded — the runtime turns that into `aborted_turn_limit` finalization (issue 09).
7. All reads go through quarantine-aware `readJson`; a quarantined session surfaces as `E_CORRUPT_SESSION` only when directly addressed by id.

## Acceptance Criteria

- [ ] Round-trip: create → save ×3 → load equals last save (deep equal), file mode 0600.
- [ ] Stale-abandon: fixture with old `updatedAt` flips to `abandoned` exactly once and persists.
- [ ] Pruning: create 205 finished fixtures → oldest 5 deleted, actives untouched; 505 replays → 5 deleted.
- [ ] `assertAppendable` triggers at the documented caps (test with synthetic big logs).
- [ ] Replay extract from a finished fixture validates against `zReplayFile` and preserves log ordering.
- [ ] Corrupt session id lookup → `E_CORRUPT_SESSION`; corrupt file skipped in `list`.

## Validation

- `test/unit/storage/sessions.test.ts`, `replays.test.ts`; temp-dir only.

## Dependencies

- 03, 05.

## Non-goals

- Runtime finalization logic (09), profile updates (07), replay frame recompute (09/22).

## Design References

- DESIGN.md §7.1, §7.4, §9.2–§9.5; ADR-004
