# Title

Integration test harness: in-memory MCP client + golden-transcript helper

## Summary

Implement `test/harness.ts`: one shared way to boot the full server (real runtime, real temp-dir storage, in-memory MCP transport) and drive it tool-by-tool; plus `playTranscript` for seeded golden-game tests used by every game issue.

## Context

DESIGN.md §19.3 mandates that all tool-level behavior is tested through the MCP boundary, not internals. Wave-1+ game issues (15–17, 26, 29–31, 36) and the determinism suite (40) consume this harness, so its API must be stable and pleasant.

## Scope

- `createHarness(opts?: {games?: GameDefinition[]}) → h` with:
  - `h.call(tool, args)` → parsed `structuredContent` (throws a typed assertion error if the tool returned an `error` object, unless `h.callRaw` is used);
  - `h.client` (raw SDK client), `h.dataDir`, `h.close()`.
- `playTranscript(h, {gameId, options, steps, finalExpect?})`: starts a game (`options` must include an explicit `seed` — the helper throws if absent, forcing deterministic transcripts), applies `steps: [{action, expect: "accept" | "reject" | {rejectCode}}]` sequentially, then asserts optional `finalExpect: {result?, reason?, scoreExact?, scoreMin?}` against the outcome; returns `{observation, outcome, sessionId}`.
- Fixture stub games (`stub-solo`, `stub-npc`) moved here from issues 09–11 tests as shared exports.
- Vitest config wiring so `test/integration/**` runs in CI (already in scaffold; verify).

## Detailed Requirements

1. Uses SDK `InMemoryTransport.createLinkedPair()`; boots via the same `createMcpServer` factory as production (no test-only server code paths).
2. Every harness instance creates its own fresh `fs.mkdtemp` data dir; `h.close()` disconnects and removes **only that directory it created** (never a caller-supplied path — there is no data-dir parameter, by design).
3. `playTranscript` step mismatch errors print the step index (0-based), the server's error object, and the current render (debuggability is the point).
4. Snapshot helpers: `h.snapshotOutcome(x)` normalizes exactly two patterns anywhere in the serialized value — ISO-8601 timestamps (`\d{4}-\d{2}-\d{2}T[0-9:.]+Z`) → `<TS>`, and `(sess|rep)_[0-9A-HJKMNP-TV-Z]{26}` → `<SESS>`/`<REP>`; nothing else is touched (gameIds/playerIds stay).
5. Refactor the tests written in issues 10–12 onto the harness (keep coverage identical; this is in-scope here to avoid drift).

## Acceptance Criteria

- [ ] Harness boots in < 1 s; 20 sequential harness creations in one vitest file don't leak handles (vitest closes cleanly).
- [ ] `playTranscript` runs a scripted stub-npc game to a win with per-step expectations, and a rejection-path script with `{rejectCode}` assertions.
- [ ] Issues 10–12 test files import the harness (no duplicated in-memory boot code remains).
- [ ] Normalized snapshots contain no timestamps/ULIDs (regex assert inside the helper's own test).

## Validation

- `test/integration/harness.test.ts` self-test + green refactored suites.

## Dependencies

- 11, 12.

## Non-goals

- Real-game fixtures (each game issue brings its own), determinism CI suite (40), security suite (39).

## Design References

- DESIGN.md §19.1–§19.3
