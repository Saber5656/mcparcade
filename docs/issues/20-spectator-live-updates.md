# Title

Spectator live updates: SSE endpoint, in-process bus subscription, cross-process fs.watch

## Summary

Implement `src/spectator/sse.ts` + `watch.ts` per DESIGN.md §11.5: the `/api/events` SSE stream fed by the runtime's in-process bus and by filesystem watching of `sessions/`/`replays/` so games driven by *other* mcparcade processes also appear live.

## Context

Live spectating is the wave-2 payoff. SSE (not WebSocket) per ADR-006. Cross-process visibility matters because users often run Claude Code and Claude Desktop simultaneously against the same data dir (§9.3).

## Scope

- `GET /api/events` (token-via-query + Host/Origin rules from issue 19; §11.2): `text/event-stream`, heartbeat comment every 25 s, events per §11.5 framing — `event: <type>` + `data: {"type", "sessionId"|"replayId", "gameId", "turn"?}` for types `session-updated`, `session-finished`, `replay-added`.
- Bus subscription: runtime bus (issue 09) → SSE broadcast.
- `watch.ts`: `fs.watch` on the two dirs, 300 ms debounce per file. On a debounced file event for an exact-pattern name (`sess_*.json` / `rep_*.json`): load the file through the quarantine-aware store reader (issues 05/06); on successful read emit the SSE event with `gameId`/`turn` from the parsed summary; on failed read (corrupt/foreign JSON) skip silently apart from the store's own quarantine warning — never crash, never emit (T9).
- Dedupe against bus-origin events: key `kind:id:turn`, suppression window = the 300 ms debounce window; `session-finished` is never suppressed by a prior `session-updated` for the same id.
- Graceful shutdown: close all SSE connections on `close()`.

## Detailed Requirements

1. SSE framing: `event: <type>\nid: <monotonic counter>\ndata: <json>\n\n`; flush immediately (`res.write`); disable Nagle where applicable. No `Last-Event-ID` replay in v1 (client refetches lists on connect).
2. Data-light rule (§11.5): events carry ids and turn numbers only — never state or renders; the SPA refetches.
3. Connection cap: 8 concurrent SSE clients; the 9th gets 503 `{"error":"too_many_streams"}` (local UI, not a broadcast service).
4. `fs.watch` caveats (U5): wrap in try/catch; if `fs.watch` throws or emits `error` (platform edge), fall back to `fs.watchFile`-style polling every 2 s and log a one-line warning; expose the active mode via `/api/meta`'s `watchMode: "watch"|"poll"` field (§11.4).
5. Filename filter: only `sess_*.json` / `rep_*.json` exact-pattern files trigger events; `.tmp.`, `.corrupt-`, `.lock` ignored.
6. Debounce map is bounded (LRU 256 entries) so a hostile flood of file events can't grow memory.

## Acceptance Criteria

- [ ] Harness-driven `act` on a live session produces a `session-updated` SSE event within 100 ms (in-proc bus path), observed by a real EventSource-style client in the test.
- [ ] Writing a session file directly into the data dir (simulating another process) produces `session-updated` within 1 s (watch path); tmp/lock/corrupt files produce nothing.
- [ ] Duplicate suppression: bus + watch for the same save yields exactly one event within the window.
- [ ] Heartbeat arrives on an idle stream (test with 1 s heartbeat override).
- [ ] 9th concurrent stream → 503; closing one admits a new one.
- [ ] `close()` ends all streams; no open-handle leaks (vitest clean exit).

## Validation

- `test/unit/spectator/sse.test.ts` with real HTTP + temp data dirs; poll-fallback path forced by injecting a throwing `watchFactory`.

## Dependencies

- 06, 19.

## Non-goals

- SPA consumption (21), replay playback (22), WebSocket anything.

## Design References

- DESIGN.md §11.5, §9.3, §21 (U5); ADR-006
