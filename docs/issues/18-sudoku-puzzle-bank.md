# Title

Sudoku puzzle bank: offline generator tool + committed bank data

## Summary

Implement `tools/gen-sudoku-bank.ts` (repo tooling, not shipped) and commit `content/sudoku/bank-v1.json` with uniqueness-verified puzzles: ≥ 60 each for easy/normal, ≥ 40 for hard (DESIGN.md §12.4; the lower hard floor reflects generator-yield unknown U3).

## Context

Runtime generation with uniqueness checking is slow and jittery; a committed bank keeps the game instant and deterministic. The generator is maintainer tooling run once per bank version.

## Scope

- Generator: full-grid backtracking generation → clue removal with uniqueness re-verification (solution counting with early exit at 2) → difficulty classification by clue count (§12.4: easy ≥ 38, normal 28–32, hard 24–26).
- Bank JSON schema: `{schemaVersion: 1, generator: {seed, generatedAt, tool, version}, puzzles: [{id, difficulty, givens: "81 chars 0-9", solution: "81 chars 1-9", clues}]}` with ids `sudoku-<difficulty>-<3-digit>` (the `generator` object is the provenance record — JSON has no comments).
- CLI: `npx tsx tools/gen-sudoku-bank.ts --per-difficulty 60 --seed <hex> --out content/sudoku/bank-v1.json` (seeded via issue 04's rng so the bank is regenerable).
- The committed bank file itself (run the tool, commit output).

## Detailed Requirements

1. Solver used for uniqueness: bitmask-based backtracking with MRV heuristic; must count solutions with early termination at 2; target < 50 ms/puzzle verification.
2. Every emitted puzzle: exactly one solution (verified), `solution` solves `givens`, clue count matches its difficulty band, grid valid.
3. Targets: `--per-difficulty` defaults to `easy=60,normal=60,hard=60` but the committed bank must meet the floors 60/60/40 (§12.4). Hard-band generation may be slow/low-yield (U3): the tool logs attempts/yield; if hard stalls below 60, commit what it produced (≥ 40) with a logged warning and note the shortfall against U3 in the PR.
4. Determinism: same `--seed` → same bank (single rng stream, documented draw order).
5. A vitest suite validates the *committed* bank (not the generator): schema, uniqueness re-verification of the first 10 puzzles per difficulty by id order (deterministic selection), clue bands, id uniqueness. This test runs in CI forever, guarding manual edits.
6. Generator has zero runtime imports from `src/` except `core/rng` (keeps tool honest and reusable).

## Acceptance Criteria

- [ ] `bank-v1.json` committed with ≥ 60/60/40 puzzles (easy/normal/hard) and the `generator` provenance object.
- [ ] CI bank-validation suite green (schema, deterministic uniqueness spot-check, clue bands, id uniqueness).
- [ ] Regeneration with the seed recorded in `generator.seed` reproduces the committed `puzzles` array byte-for-byte (`generatedAt` excluded from the comparison).
- [ ] Generation run evidence in the PR: command line, wall time, and yield log for all three difficulties (guideline: easy+normal ≤ 10 min on the maintainer machine; hard time-boxed and reported).

## Validation

- `test/unit/content/sudoku-bank.test.ts` (validates committed data); manual: regeneration run recorded in the PR description.

## Dependencies

- 01, 04 (the generator seeds through `core/rng`).

## Non-goals

- Runtime generation, difficulty via technique analysis (clue-count proxy is v1), the game plugin (17).

## Design References

- DESIGN.md §12.4, §21 (U3)
