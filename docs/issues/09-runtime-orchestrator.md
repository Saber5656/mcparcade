# Title

Game runtime: act loop, NPC resolution, limits, finalization, replayToTurn

## Summary

Implement `src/core/runtime.ts` — the orchestrator that owns sessions end to end: `startGame`, `observe`, `act` (with the normative §7.2 algorithm), universal actions, limits enforcement, finalization (stats + rating + replay), and the deterministic `replayToTurn` utility.

## Context

This is the heart of the server. Every game plugin runs inside it; every MCP tool (issues 11–12) is a thin facade over it. The §7.2 algorithm and §7.4 limits are normative — implement them exactly.

## Scope

- `Runtime` class constructed with `{registry, storage(session/replay/profile stores), rating}`.
- `startGame(gameId, options) → {session, observation}`: validate options (game schema + universal `seed`), enforce active-session cap, build rng system, call `init`, run the NPC-resolution loop (same as act step 6) so an NPC that moves first is reflected in the first observation, persist.
- `act(sessionId, rawAction) → {accepted, events, observation, outcome?}` per DESIGN.md §7.2 steps 1–9, including universal `{"type":"resign"}` and the NPC loop with its 200-step guard.
- `observe(sessionId, verbose)`.
- Finalization (§7.6): outcome, `rated` determination, profile update under lock, replay extract, `session-finished` bus event.
- Rng stream ownership: create streams `init`/`game`/`npc:<id>` once per loaded session lifetime; **streams must be reconstructed deterministically on process restart** — see requirement 6.
- `replayToTurn(replayFile, t) → {state, events}` by re-running init + actionLog[0..t) with fresh streams.
- In-process event bus (`session-updated`, `session-finished`) consumed later by the spectator (issue 20).

## Detailed Requirements

1. Follow §7.2 exactly. On NPC `act` rejection or NPC-loop overrun (§7.5 freeze semantics): persist `frozen: true` on the session (a required schema field defaulting to false — issue 06 owns the schema) and return `E_INTERNAL`. Frozen-session contract thereafter (normative): `act` → error `E_INTERNAL` with `hint: "session is frozen after an internal error and is read-only; start a new game"`, zero writes; `observe` → succeeds read-only; `list_sessions` shows it with status `active` and summary "frozen".
2. Rejected agent actions: no persistence write at all; consecutive-rejection counter lives in memory keyed by sessionId (reset on accept), driving the §7.3 escalating hint.
3. **Deterministic stream reconstruction**: rng streams cannot be checkpointed as state, so on every `act` the runtime rebuilds streams by replaying: maintain `rngCursors: {stream: count}` in the session file — each stream records how many draws it has consumed; on load, fast-forward fresh streams by that count. Plugins must draw only via the passed `Rng`. Update `rngCursors` atomically with the state write. (This makes `stateSnapshot` + cursors a valid resume point while the action log stays the audit source, ADR-004.)
4. Universal resign: before turn 5 in winloss games → `abandoned` (unrated); otherwise game's resign semantics via `terminal` shortcut: construct outcome `{result:"loss", reason:"resign"}` for winloss, `{result:"loss", reason:"resign", score:0}` for score games (escape rule §14.6), without calling `game.act`.
5. Limits (§7.4) split as follows — turn/event/file-size caps checked in the runtime before applying an agent action; breach finalizes `aborted_turn_limit` (rated=false) and still returns a final observation. The 8 KiB whole-action cap is enforced at the tool facade (issue 11); the **1 KiB free-text cap is enforced here**: after schema validation, walk the action object and reject (`E_INVALID_ACTION`) if any string leaf exceeds 1 024 bytes UTF-8 (single chokepoint for all games).
6. `maxTurns` counts global turns `t` (each accepted action, agent or NPC, increments `t` — the per-game budgets in §12–§15 were chosen for this definition). **`actionLog` contents (normative for replay):** one entry per accepted action, including NPC actions and universal actions like `{"type":"resign"}`, each `{t, playerId, action, at}`; rejected actions are never logged. `replayToTurn(replay, t)` re-runs `init` + `actionLog` entries with `entry.t ≤ t` through `game.act` with freshly derived streams.
7. Timestamps come from an injected `Clock` (`now(): Date`) so tests are reproducible; game plugins never see the clock.
8. Bus: tiny typed EventEmitter wrapper; emissions are fire-and-forget and must never throw into the act path.
8b. Cross-process safety (§9.3): every session mutation path (`act`, finalization, stale-abandon rewrite) runs inside `withLock(sessionId)` (issue 05), scoped to the load-mutate-persist span only; the lock is released before bus emission.
9. Structured stderr logging per §7.6 with control-char stripping helper `sanitizeForLog`.

## Acceptance Criteria

- [ ] Stub-game tests (a scripted `GameDefinition` fixture with an NPC) prove: NPC chain resolves within one `act`; events are filtered per visibility; rejected action mutates nothing (byte-identical session file).
- [ ] Resign before/after turn 5 produces `abandoned` vs `loss`; rating applied only for rated outcomes (spy on rating module).
- [ ] Cap breaches: turn cap, event cap, file-size cap each finalize `aborted_turn_limit` exactly once — observable as exactly one replay file written, exactly one profile update (spies), and a session file that validates as `finished`.
- [ ] Free-text cap: an action containing a 1 025-byte string leaf → `E_INVALID_ACTION`, no writes; 1 024 bytes passes.
- [ ] Logging: a full act cycle logs only to stderr with `info|warn|error` prefixes; `sanitizeForLog` strips `\x00-\x1f` (except `\n` folded to `\\n`) and `\x1b` — fixture with ANSI-escape-laden action strings asserts clean stderr capture.
- [ ] NPC misbehavior (fixture NPC returns illegal action / infinite alternation) → session frozen, `E_INTERNAL`, subsequent `act` → `E_SESSION_FINISHED`-like rejection (`E_INTERNAL` with frozen hint), `observe` still works.
- [ ] Restart determinism: play 10 turns, reload store into a fresh Runtime (fresh streams fast-forwarded via `rngCursors`), continue 10 turns → identical to 20 turns in one process (stub game uses rng heavily to make drift detectable).
- [ ] `replayToTurn` over a finished stub-game session reproduces the stored final state exactly at `t = last`.
- [ ] Session cap: 21st `startGame` → `E_TOO_MANY_SESSIONS`.

## Validation

- `test/unit/core/runtime.test.ts` (stub-game fixtures); determinism restart test doubles as the seed for issue 40's suite.

## Dependencies

- 03, 04, 06, 07, 08.

## Non-goals

- MCP wiring (10–12), spectator consumption of the bus (20), real games.

## Design References

- DESIGN.md §6, §7 (normative), §8, §9.3, §10; ADR-004
