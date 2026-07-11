# Title

Shogi game plugin: observe/act/render, outcomes, rating hookup

## Summary

Implement `src/games/shogi/index.ts` — the `GameDefinition` gluing rules (24) and engine (25) into the runtime per DESIGN.md §13: options, USI move action with reason-coded hints, kanji board render, sennichite/perpetual outcomes, and rated finalization vs difficulty anchors.

## Context

First winloss + NPC game through the full pipeline; proves rating updates, NPC loop, and resign semantics end to end.

## Scope

- `GameDefinition` `id: "shogi"`, `category: "board"`, `scoring: "winloss"`, `maxTurns: 400`, `version: "1.0.0"`.
- Options: `{difficulty?: "easy"|"normal"|"hard" = "normal", yourSide?: "black"|"white"|"random" = "black", seed?}`; `random` resolves via `init` rng; NPC gets `npcProfile: difficulty`.
- Action `{"type":"move","usi": string}` with a bounded zod schema (T1): `usi: z.string().max(6).regex(/^([1-9][a-i][1-9][a-i]\+?|[PLNSGBR]\*[1-9][a-i])$/)` — syntactic legality only; semantic legality is the adapter's job. (+ runtime resign per §7.1: before turn 5 ⇒ `abandoned` unrated, later ⇒ loss `reason: "resign"`.)
- `npcAct` → `chooseMove(state, difficulty, rng("npc:p2"))`.
- Render per §13.6; `renderWeb` model (normative, consumed by issue 27): `{kind:"shogi", board: [{file: 1-9, rank: 1-9, piece: "P"|"+P"|…|"K", owner: "black"|"white"}] (occupied squares only; USI coordinates — file counted from the right, rank from the top), handsBlack: {P: n, …}, handsWhite: {…}, lastMove: {file, rank} | null, check: boolean, checkedKing: {file, rank} | null}`.

## Detailed Requirements

1. Observation `state`: `{sfen, yourSide, sideToMove, hands, inCheck, history: last 10 [{usi, kanji}], moveNumber}`; `legalActions` list mode when ≤ 64 else schema mode with count; `verbose` always lists all.
2. Rejection mapping from adapter reason codes to hints — every code from issue 24 gets a mapping: `nifu` → "you already have an unpromoted pawn on file X"; `into_check` → "that move leaves your king in check"; `uchifuzume` → "pawn-drop checkmate is illegal"; `occupied` → "square X is occupied"; `not_in_hand` → "you have no <piece> in hand"; `blocked`/`no_piece`/`wrong_side`/`drop_rank`/`bad_promotion` → one-line explanations; schema-level syntax failure → format reminder with 3 example moves. Include ≤ 10 sample legal moves in every rejection.
3. Terminal mapping: checkmate → win/loss (`reason: "checkmate"`); `repetitionStatus() == "draw"` → draw; `perpetual_check_loss:<side>` → that side loses (`reason: "perpetual_check"`); resign → per §7.1 (abandoned before turn 5, else loss); turn cap → `aborted_turn_limit` rated=false (runtime).
4. `ratingDelta` in outcome from the runtime's rating step (anchor by difficulty); abandoned/turn-limit outcomes are unrated (`rated: false` via issue 08's predicate).
5. Events: agent move `{kind:"move", summary:"☗７六歩"}`, NPC `{kind:"npc_move", summary:"△３四歩"}`, check events `{kind:"info", summary:"check"}`.
6. rulesDoc: USI cheat-sheet (move/promote/drop syntax), scoring/anchors table, sennichite & uchifuzume explanations targeted at LLM players, 3 worked examples.
7. First move: if the agent is white, the NPC (black) moves during `start_game` — this is the runtime's post-`init` NPC-resolution loop (DESIGN.md §7.2, implemented by issue 09); this plugin only needs `currentPlayer` to report the NPC correctly at init time.

## Acceptance Criteria

- [ ] Golden transcripts (harness, each from a fresh profile): (a) a seeded winning line vs easy — recorded once during implementation with the deterministic engine, frozen as the fixture → win with `ratingDelta: +8` and rating exactly 1000→1008 (K=32 vs anchor 800 per issue 08 math); (b) resign at move 6 vs easy → loss, rating exactly 1000→976 (−24); (c) resign at move 3 → `abandoned`, unrated, rating stays 1000.
- [ ] Agent-as-white: `start_game` returns an observation where NPC's first move is already in events/history.
- [ ] Rejections: nifu, into-check, drop-on-occupied, bad syntax each produce the mapped hint + samples; state unchanged.
- [ ] Sennichite fixture (scripted 4-fold repetition) → draw outcome, rating 0.5 applied.
- [ ] Render snapshot: fixed midgame SFEN renders the documented kanji board exactly (golden string).
- [ ] Determinism: full golden game replays identically (feeds issue 40).

## Validation

- `test/integration/games/shogi.golden.test.ts` + unit tests for rejection mapping and terminal mapping.

## Dependencies

- 09, 14, 24, 25.

## Non-goals

- Spectator board rendering (27), tsume/handicap (v2), engine changes (25).

## Design References

- DESIGN.md §13 (all), §10.2, §7.2; ADR-002
