# Title

Deterministic seeded RNG: splitmix64 stream derivation + xoshiro128** with published test vectors

## Summary

Implement `src/core/rng.ts`: a master-seed → named-stream RNG system that makes every game and NPC decision reproducible across platforms and Node versions (DESIGN.md §8).

## Context

Determinism is a design pillar (ADR-004): replays, ratings integrity, and CI golden tests all depend on this module being exact and stable forever. The algorithms are fixed by design — do not substitute different PRNGs.

## Scope

- `createRngSystem(masterSeedHex: string)` → `{ stream(tag: string): Rng }`.
- `Rng`: `next(): number` in [0,1), `int(loIncl: number, hiExcl: number)`, `pick<T>(arr: T[]): T`, `shuffle<T>(arr: T[]): T[]` (Fisher–Yates, in-place copy).
- Seed parsing/normalization and the seed generator for `start_game` (`crypto.randomBytes(8)` → 16 hex chars).

## Detailed Requirements

1. Master seed: hex string `^[0-9a-f]{1,16}$`, left-padded to 64 bits. Stream seed = `splitmix64(masterSeed XOR fnv1a64(tag))`; two successive splitmix64 outputs form the four 32-bit words of a **xoshiro128\*\*** state (all-zero state guarded by re-hashing).
   Exact algorithm constants (normative — independent implementations must produce identical vectors):
   - `fnv1a64(tag)`: offset basis `0xcbf29ce484222325n`, prime `0x100000001b3n`, over the UTF-8 bytes of the tag, all mod 2^64.
   - `splitmix64` step: `state += 0x9e3779b97f4a7c15n`; `z = state; z = (z ^ (z >> 30n)) * 0xbf58476d1ce4e5b9n; z = (z ^ (z >> 27n)) * 0x94d049bb133111ebn; return z ^ (z >> 31n)` (all mod 2^64).
   - State mapping: `out1 = splitmix64()`, `out2 = splitmix64()`; `s0 = low32(out1)`, `s1 = high32(out1)`, `s2 = low32(out2)`, `s3 = high32(out2)`.
   - xoshiro128\*\* step: `result = rotl(s1 * 5, 7) * 9` (uint32); `t = s1 << 9; s2 ^= s0; s3 ^= s1; s1 ^= s2; s0 ^= s3; s2 ^= t; s3 = rotl(s3, 11)` — all uint32 (`>>> 0`), `rotl(x,k) = (x << k) | (x >>> (32−k))`.
2. Implement with `BigInt` for the 64-bit derivation, then 32-bit unsigned math (`>>> 0`) inside xoshiro128** — no floating-point anywhere except the final `next()` = `result / 2**32`.
3. `int(lo, hi)` uses rejection sampling (no modulo bias). Contract: both args must be safe integers with `lo < hi` and `hi − lo ≤ 2^32`, else **throw `TypeError`** (programmer error, not a game error). Negative bounds allowed. `pick` of an empty array throws `TypeError`. `shuffle` returns a **new array** (input never mutated), Fisher–Yates descending `i = n−1..1` with `j = int(0, i+1)`, consuming exactly `n−1` `int` calls (documented, tested — replay stability).
4. Stream tags used by the runtime are literals: `"init"`, `"game"`, `"npc:<playerId>"`. Same tag twice returns *independent copies starting from the same state* — document that callers must hold one instance per stream for the session's lifetime; the runtime owns stream instances (issue 09).
5. Publish test vectors in the test file: for master seed `deadbeefcafef00d`, the first 10 `next()` values (recorded as the exact decimal strings produced by JavaScript `String(x)` — JS number→string round-trips exactly) and the first 10 `int(0,100)` values, for each of the streams `init`, `game`, `npc:p2`. Compute once with the reference implementation, hard-code, and never change.
6. Zero dependencies; no `Math.random` anywhere in `src/` (grep test).

## Acceptance Criteria

- [ ] Test vectors pass on Node 20 and 22, ubuntu and macos (CI matrix).
- [ ] `int` distribution sanity: 100k draws of `int(0,4)` each bucket within ±2% of 25%.
- [ ] `shuffle` of `[1..52]` with a fixed seed equals a hard-coded expected permutation.
- [ ] Grep test: no `Math.random` in `src/**`.
- [ ] Rejection of malformed seeds (`E_INVALID_OPTIONS` semantics: uppercase hex, >16 chars, empty, non-hex).
- [ ] Contract violations throw `TypeError`: `int(1,1)`, `int(0.5,2)`, `int(0, 2**33)`, `pick([])`; `shuffle` leaves its input array unmodified (deep-equal before/after).

## Validation

- `test/unit/core/rng.test.ts` (vectors, distribution, shuffle, seeds) + `test/determinism/rng-vectors.test.ts` marked to run in the CI matrix.

## Dependencies

- 01.

## Non-goals

- Runtime stream ownership/wiring (09). Cryptographic randomness properties (this is a game PRNG; tokens use `crypto` directly).

## Design References

- DESIGN.md §8; ADR-004
