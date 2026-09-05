# Title

Shogi NPC engine: node-capped negamax with integer eval and three difficulties

## Summary

Implement `src/games/shogi/engine.ts` per DESIGN.md §13.5: deterministic negamax + alpha-beta over the rules adapter (issue 24), node-capped iterative deepening, integer-centipawn evaluation, and difficulty presets easy/normal/hard.

## Context

The NPC is the agent's sparring partner and the anchor for ratings (§10.2). Determinism (node caps, not wall-clock; integer eval; stable ordering; seeded tie-breaks) is non-negotiable — replays must reproduce the NPC's moves bit-for-bit (ADR-004).

## Scope

- `chooseMove(state, difficulty, rng): string` (USI) — pure given inputs.
- Search: negamax + alpha-beta, iterative deepening bounded by `{maxDepth, nodeCap}` per difficulty (easy 2/5 000, normal 4/50 000, hard 6/400 000); depth-preferred result = deepest fully completed iteration (never partial-iteration results — determinism).
- Move ordering (per §13.5, normative): captures ordered by MVV-LVA, then all remaining moves in the adapter's lexicographic USI order. No other tiers.
- Eval (integer centipawns): material table from §13.5 (P90 L315 N405 S540 G540 B855 R990; promoted = gold-equivalent except +B 1445 / +R 1550; hand pieces face value); piece-square term defined by **formula, not tables** (so it is reproducible without inventing constants): `pst(piece, file, rank) = w(piece) × (4 − max(|file−5|, |rank−5|)) + adv(piece, rank)` where `w` = {P:2, L:2, N:3, S:4, G:4, B:5, R:5, K:0, promoted: 4} and `adv` = +6 per rank advanced beyond its starting side for unpromoted pawns, else 0; mirrored for white. King safety −40 per enemy attacker within the 5×5 box around own king, tempo +20 for side to move.
- Tie-breaks (per §13.5): easy — rng-pick among moves within 30 cp of best; normal/hard — rng-pick among **exact-equal** top scores (the `npc:p2` stream is consulted in both cases; with a unique best move the pick is over a singleton).

## Detailed Requirements

1. Node accounting: the counter increments immediately before each node consumption (an `evaluate` call or an internal expansion); the cap is **pre-checked** — if `count == cap` the iteration aborts before consuming, so the counter can reach but never exceed the cap (deterministic — same count on every platform). Aborted iterations are discarded; the result is the deepest fully completed iteration.
2. Quiescence: captures-only extension up to 4 plies at the leaf (bounded; counts against the node cap). Keep it simple; no transposition table in v1 (determinism simplicity > strength).
3. Checkmate/stalemate scores: mate = ±(30000 − ply) (prefer faster mates); no legal moves and not in check does not occur in shogi (always a move exists or it's mate) — assert.
4. `chooseMove` must never return an illegal move (round-trip through `apply` in a debug assert).
5. Export `selfPlay(difficultyA, difficultyB, seed, maxPlies)` test helper (used for calibration + issue 40 fixtures).
6. Calibration (resolves U2): run easy-vs-normal and normal-vs-hard self-play, 20 games each with seeds 1..20 — the stronger side plays black on odd seeds and white on even seeds (color-balanced); draws score 0.5; the stronger side must score ≥ 65%. Record results in the PR. If not met, adjust node caps/eval constants (constants only — no algorithm change) and re-run.
7. Perf gate: on the CI runner, normal ≤ 3 s and hard ≤ 15 s per move (3× margin over the §13.5 laptop targets to avoid flaky CI) measured at the position reached by ply 20 of the seed-1 normal-vs-normal self-play game (deterministic fixture); the *hard* determinism gate is the node cap itself.

## Acceptance Criteria

- [ ] Self-verifying tactics fixtures (no external truth needed): (a) mate-in-1 — build positions where ≥ 1 legal move yields `isCheckmate` (constructed programmatically and frozen as SFEN fixtures); every difficulty must pick a mating move; (b) hanging piece — fixture with an undefended rook capturable at no material cost: normal and hard capture it; (c) mate-in-3 — from a frozen winning fixture, hard (attacker) vs normal (defender) self-play reaches `isCheckmate` within ≤ 5 plies.
- [ ] Determinism: `chooseMove` on 50 random-play positions (seeded) returns identical moves across two runs and across ubuntu/macos CI.
- [ ] Tie-breaks: easy's choice stays within 30 cp of best (verify by scoring the chosen move); a crafted position with ≥ 2 exact-equal best moves shows normal consuming the rng (spy) and choosing deterministically per seed.
- [ ] Node caps respected: instrumented counter `≤ cap` after every `chooseMove` (hard bound, pre-check semantics).
- [ ] Calibration table in PR: ≥ 65% color-balanced win-rate ordering holds for both pairings.
- [ ] Integer-only eval: `Number.isInteger` holds for the eval of 100 seeded random positions (test), plus code-review checklist confirmation that eval/search use no `/` without `Math.trunc`/`| 0` and no float literals.

## Validation

- `test/unit/games/shogi-engine.test.ts` (tactics fixtures, determinism, caps) + calibration script `tools/shogi-calibrate.ts` run manually, results pasted into the PR.

## Dependencies

- 04, 24.

## Non-goals

- Opening books, transposition tables, pondering, external engines (v1 non-goal), plugin wiring (26).

## Design References

- DESIGN.md §13.5, §8, §10.2, §21 (U2); ADR-002, ADR-004
