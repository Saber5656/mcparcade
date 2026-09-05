# Title

Escape scenario schema, validator library, and `validate-scenario` CLI command

## Summary

Implement `src/games/escape/schema.ts` + `validate.ts` and wire the real `mcparcade validate-scenario <file>` CLI command per DESIGN.md §14.2–§14.3: the normative v1 scenario format, zod + referential validation, caps, and author-friendly diagnostics.

## Context

The scenario format is the escape game's public contract (bundled content, community content, the authoring guide all depend on it). It is also a parser security boundary (T7): user files must be fully validated before the engine ever sees them.

## Scope

- zod schema for the full §14.2 format. Per-kind payloads (normative — §14.2 defers exact fields to this issue):
  - Conditions: `flag {flag, is: boolean}` · `has_item {item}` · `entity_state {entity, state}` · `in_room {room}` · `turns_at_least {turns: int ≥ 0}`.
  - Effects: `set_flag {flag, value: boolean}` · `add_item {item}` · `remove_item {item}` · `reveal_item {item, room}` · `set_entity_state {entity, state}` · `unlock_exit {room, dir}` / `lock_exit {room, dir}` (clears/sets the flag named by that exit's `lockedBy`) · `move_player {room}` · `show_text {text}` · `end_game {ending}`.
  - Unknown keys are rejected everywhere (`.strict()`), with one tolerance: a string field named `_comment` is allowed (and ignored) at any object level, for authoring notes (coordinated with issue 42's starter template).
- Referential validator: unknown room/entity/flag/item/ending references; duplicate ids; `start.room` exists; **item semantics** (§14.2): every id used as an item (`has_item`, `add_item`, `remove_item`, `use.item`, `interaction.with`, `start.inventory`) must reference an entity with `portable: true` (error otherwise); win-ending static reachability (graph walk treating conditions as satisfiable); orphan warnings (unreachable rooms, unused flags — warnings, not errors).
- Caps (§14.3) enforced with precise messages: file ≤ 256 KiB, rooms ≤ 100, entities ≤ 500, interactions/entity ≤ 20, effects/interaction ≤ 20, text ≤ 2 KiB, hints ≤ 10.
- Two layers: `validateScenario(json: unknown) → {ok, errors: [{level, path, message}], scenario?}` (pure, no fs — enforces every cap except file size) and `validateScenarioFile(path) → same + fileErrors` (does `stat` size check *before* reading, then read + `JSON.parse` + `validateScenario`; used by the CLI and by issue 32). CLI: pretty text output (or `--json`), exit 0 ok / 1 errors found / 2 unreadable-or-unparseable file.

## Detailed Requirements

1. Diagnostics carry JSON-paths (`rooms[3].exits[0].to`) and human messages ("references room 'hallwy' — did you mean 'hallway'?" with nearest-id suggestion via Levenshtein ≤ 2).
2. Reachability: build a graph of rooms via exits (ignoring locks), plus effect-driven `move_player`; a win `end_game` must be triggerable from some reachable room's interaction; unreachable win → **error**; unreachable normal room → warning.
3. `once` interactions, `states.all` containing `states.current`, `hints` tiers strictly increasing, `par.turns ≥ 1` — all validated.
4. Strings are plain text: reject control characters (except `\n`) anywhere; normalize to NFC. No templating syntax is defined or interpreted (document explicitly in the schema module). T8 hardening: the schema defines **no** instruction-like metadata fields (the only free-text fields are the enumerated descriptive ones); `author` is capped at 80 chars and is *inert* — it may appear in validator output and the spectator, but must never be included in agent-facing observations (engine contract, tested in issue 29).
5. File reading (`validateScenarioFile`): `JSON.parse` failure → exit 2, printing the raw `SyntaxError.message` (no position guarantee — Node's message format is not stable across versions).
6. The validator is pure (no fs) — the CLI and issues 29/32 share it.
7. Ship 6+ invalid fixtures under `test/fixtures/escape-invalid/` and 1 minimal valid fixture — the regression corpus issue 39 reuses. Each invalid fixture's test asserts the exact expected `path` (not just "some error"): bad ref → `rooms[0].exits[0].to`; cap breach → `rooms`; dup id → `entities[1].id`; unreachable win → `endings[0]`; control chars → the specific text field path; wrong effect kind → the specific `effects[n].kind` path (fixture files authored to make these paths true).

## Acceptance Criteria

- [ ] Valid minimal fixture passes; all invalid fixtures fail with the documented `path` + message (snapshot the diagnostics).
- [ ] Nearest-id suggestion appears for the bad-ref fixture.
- [ ] Cap tests at the exact boundary (100 rooms ok, 101 error; 256 KiB ok, +1 byte error).
- [ ] CLI: exit codes 0/1/2 as specified; `--json` emits machine-readable diagnostics array; human mode prints level-colored lines (respect `NO_COLOR`).
- [ ] `_comment` fields are accepted and ignored; any other unknown key is rejected with its path.
- [ ] Reachability: fixture with win behind an exit-locked chain still passes (locks ignored); fixture with win in a room with no path fails.

## Validation

- `test/unit/games/escape-schema.test.ts` + CLI child-process tests in `test/integration/cli-validate.test.ts`.

## Dependencies

- 03, 13.

## Non-goals

- Engine semantics (29), bundled content (30/31), custom-dir loading (32), authoring guide (42).

## Design References

- DESIGN.md §14.2–§14.3, §18.2 (T7); §16 (CLI)
