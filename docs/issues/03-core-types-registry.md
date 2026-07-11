# Title

Core domain types, error taxonomy, and game registry

## Summary

Implement `src/core/types.ts`, `src/core/errors.ts`, `src/core/limits.ts`, `src/core/ulid.ts`, and `src/core/registry.ts`: the `GameDefinition` contract, observation/event/outcome shapes, error codes, global limits, ULID generation, and the static registry every game plugs into.

## Context

This is the contract layer between runtime, tools, storage, spectator, and all six games (DESIGN.md §6). Getting the shapes right here prevents churn in every later issue. No game logic lands here.

## Scope

- Types + zod schemas for: `GameDefinition<S,A>`, `GameView`, `Observation`, `GameEvent`, `ActOutcome`, `Outcome`, `Player`, `SessionStatus`, `LegalActions` (list/schema modes).
- **This issue owns every identifier referenced by the §6.1 signatures**: `GameId` (string union of the six ids), `PlayerId` (`p${number}` template type), `ValidatedOptions` (`Record<string, unknown>` post-zod), `InitResult<S>` (`{state: S, players: Player[]}`), `WebRenderModel` (discriminated union on `kind`, extensible), and the `Rng` **interface** (`next/int/pick/shuffle` — the implementation lands in issue 04 against this interface).
- `errors.ts`: `McparcadeError` class with `code` from DESIGN.md §7.5, `message`, `hint`; helper constructors per code.
- `limits.ts`: the constants table from DESIGN.md §7.4, exported as a single frozen object.
- `ulid.ts`: **monotonic** ULID (Crockford base32, 48-bit time + 80-bit randomness from `crypto`; within the same millisecond, increment the previous random tail by 1 instead of regenerating — the standard monotonic-factory rule; on tail overflow, wait for the next millisecond), plus `isValidId(kind, id)` matching `^(sess|rep)_[0-9A-HJKMNP-TV-Z]{26}$`.
- `registry.ts`: `registerGame(def)`, `getGame(id)`, `listGames()`; duplicate id registration throws at startup.

## Detailed Requirements

1. Follow DESIGN.md §6.1 signatures exactly, including `currentPlayer`, `terminal`, optional `npcAct`/`renderWeb`, and the purity contract stated as doc comments on the interface (no Date/Math.random/fs/network in plugins).
2. `Observation` and `Outcome` field names must match DESIGN.md §6.4/§6.6 exactly (camelCase, `status: "your_turn" | "finished"`).
3. `GameEvent.visibility` is `"public" | PlayerId`; provide `visibleTo(events, playerId)` filter helper.
4. Zod schemas are exported alongside TS types (`zObservation`, `zOutcome`, …) so tools and storage validate with the same source of truth.
5. Error objects serialize to `{code, message, hint}` via `toJSON()`; never include stack traces in serialized form.
6. `ulid.ts` uses `crypto.randomBytes`; ids are generated only server-side. Include a doc comment that ids are validated before any path use (DESIGN.md §18.4).
7. No I/O, no imports from `storage/` or `server/` (dependency direction: core is the bottom layer; enforce with an eslint-ish grep test in `test/unit/`).

## Acceptance Criteria

- [ ] All types compile under `strict`; zod schemas round-trip valid fixtures and reject at minimum these invalid fixtures: `Observation` with `status: "npc_turn"`; `Observation` missing `render`; `GameEvent` with `visibility: "p1x"`-style malformed id; `Outcome` with `result: "timeout"`; `Outcome` with non-numeric `score`; `ActOutcome` rejection missing `code`; a registry definition whose `id` is not a known `GameId`.
- [ ] Error taxonomy: one constructor per §7.5 code exists (table-driven test over the full code list incl. `E_REPLAY_NOT_FOUND`); `toJSON()` yields exactly `{code, message, hint}` with no `stack` key.
- [ ] `limits.ts` exports exactly the §7.4 values (table-driven equality test — 20 sessions, 8 KiB action, 1 KiB free-text, 20 000 events, 2 MiB file, 200 NPC steps, 500 replay-page entries).
- [ ] `visibleTo(events, "p2")` returns public events plus p2-private ones and excludes p1-private ones (fixture test).
- [ ] `isValidId` rejects: traversal strings (`../x`), uppercase-invalid Crockford chars (`I,L,O,U`), wrong prefixes, wrong lengths — table-driven test.
- [ ] Registry throws on duplicate registration; `listGames()` returns games in registration order.
- [ ] ULID: 1000 generated ids are unique, lexicographically non-decreasing when generated in sequence within one process, and match the regex.
- [ ] A unit test asserts `src/core/**` has no imports from `src/storage`, `src/server`, `src/spectator`, `src/games`, and contains no `path.join`/`node:path` usage (path construction is `storage/paths.ts`-only per §18.4; `runtime.ts` is exempted from the import rule for `storage/` when issue 09 lands — scope the grep to this issue's files).

## Validation

- `npm test` unit suite `test/unit/core/*.test.ts` covering every bullet above.

## Dependencies

- 01.

## Non-goals

- Runtime behavior (09), tool schemas for specific tools (10–12), any game logic.

## Design References

- DESIGN.md §6 (domain model), §7.4–§7.5 (limits, errors), §9.2 (id regex), §18.4 (validation chokepoints)
