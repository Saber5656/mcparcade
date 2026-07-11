# Title

Replay viewer: replay list, frame recompute API, step/play controls

## Summary

Implement `GET /api/replays/:id/frame/:t` (server, via runtime `replayToTurn` with an LRU cache) and the SPA replay pages per DESIGN.md §11.7: list replays, open one, step/back/play(1×/4×) through deterministic frames with the same renderer registry as live views.

## Context

Replays close the "records" pillar loop: the human reviews how the agent played. Frame recompute leans on ADR-004 determinism; the `gameVersion` gate protects against rules drift.

## Scope

- Server routes (both inherit issue 19's full pipeline: Host/Origin, GET-only, token, ULID-regex-before-store):
  - `GET /api/replays/:id` → `{meta: {replayId, sessionId, gameId, gameTitle, gameVersion, endedAt, total}, outcome, finalView: {render, web, panels}, reveal: object|null}` (final view built from the stored `finalState`; `reveal` passthrough as in issue 19).
  - `GET /api/replays/:id/frame/:t` — `t` must parse as a non-negative integer (else 400); `t > total` clamps to `total` (200); recompute via `replayToTurn` (issue 09), LRU cache 32 frames keyed `replayId:t`, `gameVersion` exact-match gate → on mismatch 409 `{"error":"version_mismatch"}` (client falls back to `finalView`).
- SPA: `#/replays` list (from `/api/replays`), `#/r/:id` viewer — render area (renderer registry or `<pre>`), event log up to current `t`, controls: ⏮ ◀ ▶ ⏭, play/pause with 1×(1 frame/s)/4× speed, keyboard ←/→/space, turn slider.
- Version-mismatch fallback UI: show stored final `stateSnapshot` render + full log with a notice (§11.7).

## Detailed Requirements

1. Frame response: `{t, total, publicView: {render, web, panels}}` — exactly the §11.6 `publicView` projection computed from the recomputed state at turn `t`, with `panels.log` = public events with `event.t ≤ t` (last 100). Frames are always the **public** projection, including for werewolf (§11.7): the reveal is a separate panel from `finalView.reveal`, spoiler-gated by issue 37.
2. Recompute cost bound: for long logs `replayToTurn` is O(t) per call — mitigate with the LRU plus nearest-cached-prefix resume. Extend the runtime (this issue): cache entries are `{t, state, rngCursors}`; `replayToTurn(replay, t, from?: {t, state, rngCursors})` reconstructs streams fast-forwarded by the cursors and applies only log entries with `from.t < entry.t ≤ t`. Correctness gate: a resumed frame must hash-equal the same frame computed from scratch (explicit test). Target: stepping forward is O(Δt).
3. Controls disable past ends; play stops at final frame and shows the outcome banner.
4. Version-mismatch fallback UI: notice element `[data-testid="version-mismatch-notice"]` with text "Replay recorded with an older game version — showing final position only", above the `finalView` render.
5. Cache is per-process, bounded, and cleared on `close()`.

## Acceptance Criteria

- [ ] Frames of a finished golden maze game: `t=0` shows the initial board; `t=total` render equals the stored final snapshot render (string equality); a resumed mid-frame hash-equals the from-scratch computation.
- [ ] Sequential forward stepping hits the prefix-resume path (assert full-replay recompute happens at most once across 10 forward steps, via spy).
- [ ] `t` handling matrix: `t=-1` → 400, `t=abc` → 400, `t=total+5` → 200 with `t: total`.
- [ ] Version gate: fixture replay with `gameVersion: "0.0.1"` → 409; SPA renders `finalView` with the `[data-testid="version-mismatch-notice"]` element (jsdom).
- [ ] Controls: keyboard and buttons move `t` correctly at bounds (jsdom unit tests on the controller module).
- [ ] Unauthorized/malformed access: no token → 401; traversal-style id → 404 with zero store access (spy) — inherited pipeline verified on the new routes.

## Validation

- `test/unit/spectator/replay-frames.test.ts` (server) + jsdom controller tests; manual smoke: replay a real 2048 game at 4×.

## Dependencies

- 06, 09, 21.

## Non-goals

- Export/share of replays (v2), werewolf reveal-view shaping (37), editing/annotations.

## Design References

- DESIGN.md §11.7, §9.5; ADR-004
