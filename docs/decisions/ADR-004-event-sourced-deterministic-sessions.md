# ADR-004: Sessions are seed + action log (event-sourced, deterministic)

Status: accepted

## Context

Records/ratings/replays are a core pillar ("the agent measurably grows"). Replays can be stored as (a) rendered frames, (b) state snapshots per turn, or (c) the inputs needed to recompute everything. Games include randomness (2048 spawns, maze generation, NPC tie-breaks).

## Decision

A session is defined by `(gameId, gameVersion, seed, options, actionLog)`. All randomness flows through named, seeded rng streams (xoshiro128** seeded via splitmix64; DESIGN.md §8). Replaying the log reproduces every state and event bit-for-bit; a `stateSnapshot` is stored **only** as a fast-resume cache, never as the source of truth. Game plugins are pure (no clock/random/I-O access).

## Consequences

- Replay files are small; the spectator recomputes any frame on demand (§11.7); a determinism CI suite replays every golden game on every platform (§19).
- Cheating/corruption is detectable (log doesn't replay ⇒ quarantine).
- Cost: strict plugin purity rules; integer-only evaluation in the shogi engine (no float drift); and a `gameVersion` compatibility gate — replays record the `gameVersion` that produced them, frame recompute (§11.7) runs only on an exact version match, and on mismatch the spectator falls back to the stored final `stateSnapshot` plus the textual log. Documented in DESIGN.md §9.5/§11.7.

## Alternatives rejected

- Snapshot-per-turn: 100× storage, still can't verify integrity.
- Non-deterministic NPCs with logged choices only: workable but forfeits the stronger invariant and cross-platform reproducibility that make ratings trustworthy.
