# Title

Sudoku game plugin: bank-backed puzzles, place/erase/check actions, silent-error scoring

## Summary

Implement `src/games/sudoku/` per DESIGN.md §12.3: puzzles selected from the committed bank (issue 18), immutable givens, rule-conflict `check` action, silent solution-error counting, and scoring.

## Context

Sudoku exercises: bundled-content loading, actions with coordinates, and a scoring model that punishes guessing without leaking the solution. The bank (issue 18) must exist first.

## Scope

- `GameDefinition` `id: "sudoku"`, `category: "puzzle"`, `scoring: "score"`, `maxTurns: 500`, `version: "1.0.0"`.
- Options: `{difficulty?: "easy"|"normal"|"hard" = "normal", puzzleId?: string, seed?}`.
- Actions: `place {row,col,digit}` (1-based), `erase {row,col}`, `check {}`, runtime resign.
- Bank loader with zod validation of `content/sudoku/bank-v1.json`. Loader-time invariants (fail startup on violation): schema shape per §12.4 (`id, difficulty, givens` 81 chars of `0-9`, `solution` 81 chars of `1-9`, `clues`, plus the `generator` provenance object), unique puzzle ids, ≥ 1 puzzle per difficulty, `solution` consistent with `givens` (every non-zero given matches), `clues` equals the count of non-zero givens. Deep uniqueness re-verification stays in issue 18's CI suite.

## Detailed Requirements

1. Puzzle selection: explicit `puzzleId` wins (unknown → `E_INVALID_OPTIONS` listing count per difficulty); else `rng("init").pick` over the difficulty's puzzles.
2. State: `given: boolean[81]`, `grid: number[81]` (0 empty), `errorsCommitted`, `checksUsed`, `turns`. `place` on a given or already-equal value → rejection; `place` contradicting the **unique solution** increments `errorsCommitted` silently (no feedback) and is accepted; `erase` of a wrong own-entry does not decrement — `errorsCommitted` counts wrong placements ever committed, not the current number of wrong cells (state this in rulesDoc). `checksUsed`/`summaryExtras: {errorsCommitted, checksUsed}` per DESIGN.md §12.3/§10.3.
3. `check`: reports only *rule* conflicts (duplicate digit in a row/col/box), never solution digits; consumes a turn; increments `checksUsed`. Output schema (normative): `{conflicts: [{kind: "row"|"col"|"box", a: {row, col}, b: {row, col}}]}` — 1-based coordinates, each unordered cell pair reported once with `a < b` in (row, col) lexicographic order, list sorted by (kind, a.row, a.col, b.row, b.col).
4. Terminal: grid full ∧ rule-valid (⇒ equals the unique solution). Score per §12.3: `max(0, 1000 − 10·errorsCommitted − max(0, turns − blanksAtStart))`; `summaryExtras: {errorsCommitted, checksUsed}`.
5. Render: 9×9 with `│ ─ ┼` box separators every 3; givens rendered as digits, player entries identically (state's `given` mask differentiates for the agent); blanks `·`. Footer: filled/blank counts, turns.
6. `legalActions`: schema mode `{summary: "place(row 1-9, col 1-9, digit 1-9) | erase(row,col) | check", count: <empty cells × 9>}`.
7. `renderWeb`: `{kind:"grid", rows:9, cols:9, cells:[{r,c,value,given}]}`.
8. rulesDoc explains scoring incl. the silent-error mechanic explicitly (agents must know mistakes cost even if invisible).

## Acceptance Criteria

- [ ] Bank loads and validates at registry init; a corrupted bank fixture fails startup with an error naming the bank file path and the first failing puzzle id (or zod issue path).
- [ ] Malformed inputs each rejected with `E_INVALID_ACTION`/`E_INVALID_OPTIONS`: `place` with row 0 / col 10 / digit 0; unknown action type; `difficulty: "extreme"`; `puzzleId` of 300 chars; `puzzleId` unknown (hint lists per-difficulty counts).
- [ ] Golden transcript on a fixed easy puzzle: scripted perfect solve → score = 1000 (turns = blanks); a solve with 2 wrong-then-erased placements scores 1000 − 20 − extraTurns.
- [ ] `check` on a fabricated conflict returns exactly the conflicting coordinate pairs; on a clean board returns "no conflicts".
- [ ] Givens immutable (rejection), erase of given rejected.
- [ ] Same seed + difficulty picks the same puzzle twice.
- [ ] For the first 10 puzzles per difficulty (by id order): filling the grid with the bank's `solution` terminates as a win, and the completed grid is rule-valid (deterministic selection, no randomness).

## Validation

- `test/unit/games/sudoku.test.ts` + golden harness transcripts (perfect and flawed paths).

## Dependencies

- 04, 09, 14, 18.

## Non-goals

- Puzzle generation (18), pencil-mark/notes actions, hint action (escape-only in v1).

## Design References

- DESIGN.md §12.3–§12.4, §10.3
