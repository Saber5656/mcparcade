# Title

Profile store and per-game stats aggregation

## Summary

Implement `src/storage/profile.ts` and `src/rating/stats.ts`: the `profile.json` schema (DESIGN.md §10.1) and the pure stats-update function applied at finalization for both winloss and score games.

## Context

The profile is the product's "the agent grows" ledger: plays, win/loss, ratings, best scores, streaks, recent replays. It is updated exactly once per finished session, under the advisory lock (§9.3), by the runtime (issue 09).

## Scope

- `zProfileFile` v1 schema per DESIGN.md §10.1 (games map keyed by gameId; winloss shape and score shape).
- `loadProfile()` (creates default on first run), `updateProfileOnFinalize(profile, sessionSummary) → profile'` (pure), `saveProfile` (locked RMW via `withLock("profile", …)`).
- Streak semantics, `byDifficulty` tallies, `recentReplays` ring (max 10, newest first).

## Detailed Requirements

1. Winloss games record `plays/wins/losses/draws/abandoned`, `rating` (written by issue 08's function — this issue stores whatever number it is given), `peakRating`, `streak` (+n consecutive wins, −n consecutive losses; draws reset to 0; abandoned/turn-limit games increment `abandoned` and `plays` but leave streak untouched). `byDifficulty` records `{w,l,d}` per difficulty for rated results only (abandoned/turn-limit excluded).
2. Score games record `plays`, `bestScore`, `avgScore` (running mean, 1 decimal), plus `lastExtras` = the most recent outcome's `summaryExtras` copied verbatim (already an optional `Outcome` field per DESIGN.md §6.6; type `Record<string, number|string|string[]>` — escape flattens par comparison into scalar keys like `turns`, `parTurns`, `ending`). No other per-game accumulation in v1.
3. `aborted_turn_limit` counts as `plays` + `abandoned` bucket for winloss games; for score games it records the score achieved.
4. `recentReplays` prepends the new replay id, dedupes, truncates to 10.
5. The update function is pure and total: unknown gameId creates the entry; missing fields default sanely; it must never throw on any `Outcome` shape that passes core schemas.
6. Concurrency: `saveProfile` re-reads inside the lock and re-applies the update to the freshest copy (RMW), so two processes finishing games simultaneously both land.

## Acceptance Criteria

- [ ] Table-driven tests for shogi (all at difficulty "normal"): sequence W W L D W → `plays:5, wins:3, losses:1, draws:1, abandoned:0, streak:+1, byDifficulty.normal:{w:3,l:1,d:1}`; then one abandon → `plays:6, abandoned:1, streak:+1` (unchanged streak/byDifficulty); then one turn-limit → `plays:7, abandoned:2`. Score sequence for g2048 `[1200, 800, 2000]` → `plays:3, bestScore:2000, avgScore:1333.3, lastExtras` = last outcome's summaryExtras.
- [ ] First-run creates a valid default profile file with mode 0600.
- [ ] Corrupt `profile.json` (truncated JSON) on load → quarantined per §9.4 and a fresh default profile is created; the finalize update then succeeds (T9 regression).
- [ ] Two concurrent finalizations (simulated with manual lock interleaving) both appear in the final file.
- [ ] `recentReplays` capped at 10, newest first, no duplicates.
- [ ] Property test: 500 random valid outcomes never throw and keep the file schema-valid.

## Validation

- `test/unit/storage/profile.test.ts`, `test/unit/rating/stats.test.ts`.

## Dependencies

- 03, 05.

## Non-goals

- Elo math (08), when-to-call decisions (09), `get_record` tool shaping (12).

## Design References

- DESIGN.md §9.3, §10.1, §10.3
