# Title

Elo rating module with fixed NPC anchors

## Summary

Implement `src/rating/elo.ts`: the pure Elo update used for winloss games (shogi, werewolf) against fixed NPC anchor ratings (DESIGN.md §10.2).

## Context

Ratings make the agent's progress legible ("my agent is 1084 at shogi"). Anchors are product constants, not human-Elo claims; determinism and exact rounding rules matter because tests snapshot rating trajectories.

## Scope

- `updateRating({rating, ratedGames}, anchorRating, score: 1|0.5|0) → {rating, ratedGames}`.
- Anchor table: shogi easy 800 / normal 1200 / hard 1600; werewolf 1200.
- Constants: start 1000, floor 400, K=32 (K=16 once `ratedGames ≥ 30`).

## Detailed Requirements

1. `expected = 1 / (1 + 10^((anchor − rating)/400))` computed in floating point; `rating' = Math.round(rating + K*(score − expected))` — JavaScript `Math.round` semantics (half rounds toward +∞) are **normative for the whole product** (DESIGN.md §10.2 defers to this issue); clamp to ≥ 400. Include a half-value tie test (e.g. a delta of exactly ±0.5 → verify +∞-ward rounding for both signs).
2. Call contract: the caller (runtime, issue 09) invokes `updateRating` **only** for rated results, gated by this module's shared predicate `isRatedResult(outcome)`: rated ⇔ `result ∈ {win,loss,draw}` ∧ `outcome.rated !== false` (`rated` is the optional `Outcome` flag from DESIGN.md §6.6, default true). `updateRating` itself unconditionally increments `ratedGames` by 1 per call.
3. Export `ANCHORS: {shogi: {easy, normal, hard}, werewolf: {default}}` frozen; `describe_game` reads these for its docs (issue 10).
4. Pure module: no I/O, no imports outside core types.

## Acceptance Criteria

- [ ] Known-value tests: (1000 vs 1200, win, K32) → 1024; (1000 vs 1200, loss) → 992; (1000 vs 800, loss) → 976; (1000 vs 1200, draw) → 1008. (Verify these by the formula before hard-coding; they must match `Math.round` semantics.)
- [ ] K switches to 16 at the 30th rated game boundary (test 29→30→31).
- [ ] Floor: rating never goes below 400 (loss streak from 410).
- [ ] `isRatedResult` truth table matches the predicate above.

## Validation

- `test/unit/rating/elo.test.ts`.

## Dependencies

- 03.

## Non-goals

- Stats/streaks (07), when ratings apply (09), any UI.

## Design References

- DESIGN.md §10.2 (rating math), §6.6 (`rated` flag), §13.7 (shogi rated/unrated cases), §15.8 (werewolf practice mode)
