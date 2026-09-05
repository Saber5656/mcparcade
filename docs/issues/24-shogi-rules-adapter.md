# Title

Shogi rules adapter over tsshogi with edge-rule acceptance tests

## Summary

Implement `src/games/shogi/rules.ts` — the only module importing `tsshogi` — exposing the `ShogiRules` interface (DESIGN.md §13.1) with verified enforcement of nifu, last-rank drops, uchifuzume, and sennichite; SFEN/USI conversion; and check/checkmate detection.

## Context

Research (docs/research/shogi-libraries.md) selected tsshogi (MIT) but flagged edge-rule coverage as **unverified** (known unknown U1). This issue resolves U1: the acceptance tests below are the contract; whatever tsshogi lacks, the adapter implements on top — tests must not be weakened to fit the library.

## Scope

- `ShogiRules` (names per DESIGN.md §13.1): `initialPosition()`, `fromSfen(sfen)`, `toSfen(state)`, `legalMoves(state): string[]` (USI), `apply(state, usi) → {state} | {error: reason}`, `inCheck(state)`, `isCheckmate(state)`, `repetitionStatus(history): "none"|"draw"|"perpetual_check_loss:<side>"`, `moveToKanji(state, usi): string` (e.g. `☗７六歩`).
- Position history tracking for sennichite (position = SFEN board+hands+side, count 4; perpetual check per §13.4).
- Acceptance-test suite with named SFEN fixtures for every rule listed below.

## Detailed Requirements

1. `legalMoves` output: full USI (`7g7f`, `8h2b+`, `P*5e`), deterministic order (sort lexicographically after generation — NPC move ordering happens in issue 25, not here).
2. Rules that MUST hold (each with ≥ 1 positive + 1 negative fixture):
   - standard piece movement & captures; promotion optional in the zone, **forced** where a piece would have no legal future move (pawn/lance to last rank, knight to last two);
   - drops: only pieces in hand, not on occupied squares; pawn/lance not on last rank, knight not on last two;
   - **nifu**: no second unpromoted pawn on a file;
   - no move may leave own king in check (incl. drops);
   - **uchifuzume**: a *pawn drop* giving immediate checkmate is illegal (non-pawn-drop mate and pawn-*push* mate remain legal — fixture both);
   - **sennichite**: 4th repetition of position+side → `draw`; if one side gave check in every intervening move of the repetition cycle → that side loses (`perpetual_check_loss`).
3. `apply` on an illegal USI returns `{error: reasonCode}` with codes: `not_a_move, no_piece, wrong_side, blocked, into_check, nifu, drop_rank, uchifuzume, occupied, not_in_hand, bad_promotion` — the plugin (issue 26) maps these to hints. When multiple reasons apply, report the first in this normative check order: `not_a_move → wrong_side → no_piece/not_in_hand → occupied → blocked → drop_rank → nifu → bad_promotion → into_check → uchifuzume` (fixtures are crafted single-reason; the precedence rule exists so overlapping cases are still deterministic).
4. If tsshogi provides a rule natively, use it and *prove it* with the fixture; if not, implement in the adapter (likely candidates per research: uchifuzume, perpetual-check attribution).
5. Wrap tsshogi types completely: nothing outside `rules.ts` mentions tsshogi (grep-test).
5b. Supply-chain hygiene (T12): `tsshogi` is already in the 4-package runtime budget (issue 01) — verify at install: exact version recorded in the committed lockfile, license field is MIT (assert in a test reading `node_modules/tsshogi/package.json`), and no new transitive runtime dependencies are introduced beyond tsshogi's own (snapshot `npm ls tsshogi --json`).
6. Record the resolved U1 findings (what tsshogi covered vs what was implemented here) in a closing comment AND update docs/research/shogi-libraries.md's risk section.

## Acceptance Criteria

- [ ] Every rule in requirement 2 has passing positive+negative fixture tests (SFEN strings with one-line comments explaining each).
- [ ] Round-trip: `toSfen(fromSfen(s)) === s` for 20 varied fixtures incl. hands.
- [ ] `legalMoves(initialPosition())` returns exactly 30 moves (known value for shogi's initial position).
- [ ] Perft sanity from the initial position: depth 2 = exactly 900; depth 3 = exactly 25 470 (published perft values; any deviation is a generation bug).
- [ ] `moveToKanji` renders 5 documented cases (moves, drops, promotions) exactly.
- [ ] Grep-test: `tsshogi` imported only in `rules.ts`.

## Validation

- `test/unit/games/shogi-rules.test.ts` — the fixture suite above; CI matrix (determinism of move ordering).

## Dependencies

- 01, 03.

## Non-goals

- Engine/eval (25), plugin observe/act (26), jishogi/impasse (v1 non-goal §3.2), handicap positions.

## Design References

- DESIGN.md §13.1, §13.4; docs/research/shogi-libraries.md (U1)
