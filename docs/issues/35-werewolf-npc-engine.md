# Title

Werewolf NPC engine: suspicion model, personas, wolf/seer strategies

## Summary

Implement `src/games/werewolf/npc.ts` per DESIGN.md §15.6: deterministic per-NPC suspicion scoring updated from public events, persona multipliers, and the decision policies — utterances, votes, wolf kill targeting (with deflection/bus rules), and seer divine/claim strategy.

## Context

This is what makes werewolf *feel* like a game rather than a dice roll (known unknown U4: believability). All logic is heuristic, integer-weighted, seeded, and constants-tunable — the weights table is the API for future tuning.

## Scope

- `SuspicionTracker`: `S[i][j]` integer matrix (i's private belief about j) updated by `updateFromEvent` implementing the §15.6 weight table exactly, **plus** one shared `publicSuspicion[j]` vector — the same weights applied with neutral persona to public events only (the "town view"; §15.6). Wolves' knowledge lives in a separate `knownWolves` set per wolf — teammates are excluded from a wolf's accuse/vote/kill target pools by rule, never encoded as S values (keeps the bus threshold meaningful). Personas are integer thresholds per §15.6 (aggressive accuses at S ≥ 2, cautious at S ≥ 5, follower never initiates; cautious claims one day later) — all constants in `npc-weights.ts`.
- Decision policies invoked by the plugin (36) at each NPC input point:
  - `npcUtterance(state, npcId, tracker, rng) → Utterance` (acts chosen by rules below; no flavor — NPCs speak acts only);
  - `npcVote(state, npcId, tracker, rng) → name`;
  - `npcNightChoice(state, npcId, tracker, rng)` (wolf kill / seer divine);
  - persona assignment at setup (rng from `npc:<id>` streams, weighted: 40% follower / 35% cautious / 25% aggressive).
- Knowledge integration (the §15.6 "+6 confirmed-by-my-own-knowledge lie" row, both cases): the real seer applies +6 to any claimant whose claim or reported result contradicts its own divinations; a wolf applies +6 to any non-teammate claimant whose reported result contradicts the wolf's `knownWolves` knowledge (e.g. "divined Casey: werewolf" when Casey is not a wolf).

## Detailed Requirements

1. Weight table (§15.6) implemented verbatim for both private `S` and the shared `publicSuspicion` (neutral persona, public events only); all deltas integers; per-day persona noise `±0–1` applies to private `S` only, drawn from `npc:<id>` streams once per NPC per day at announce, in seat order (documented).
2. Utterance policy (per discussion turn, deterministic given tracker+rng):
   - real seer: claim + report when trigger conditions hit (a **public** seer claim by someone else exists / found a wolf / day ≥ 3; `cautious` shifts to day ≥ 4 unless personally accused);
   - wolves: fake-claim seer only if no public seer claim exists yet and persona is aggressive (rng threshold 30%); otherwise accuse the deflection target — the non-wolf with the 2nd-highest `publicSuspicion` (bus rule: the partner is targetable only when partner `publicSuspicion ≥ 8`);
   - villagers: accuse `argmax S` over alive others when that max ≥ the persona threshold (aggressive 2 / cautious 5; follower never initiates and instead echoes the current plurality vote intent with `declare_vote`); `defend` an accused player when personally confident in them (own `S ≤ 1`);
   - otherwise `pass`. ≤ 2 acts per NPC utterance (claim+report counts as 2).
3. Vote policy: villagers/seer vote `argmax S` (ties: rng); wolves vote the deflection target, or pile onto the current plurality when it's not a wolf (`follower` wolves always pile); runoff: same policies restricted to the tied set.
4. Kill policy (lead wolf), strict priority: (a) a publicly claimed seer (the real one if two claim — wolves know which); (b) the non-wolf who has made the most accuse/vote acts targeting any wolf (tie → higher `publicSuspicion`); (c) rng among non-wolves.
5. Seer divine policy: highest-S unconfirmed alive; never re-divines confirmed players.
6. All tie-breaks: rng from the NPC's own stream, then seat order (total order guaranteed — no `Object.keys` iteration order reliance; sort explicitly).
7. Believability calibration (resolves U4): simulate 200 seeded games NPC-only; assert sanity bounds — village win-rate 35–65% for 7p, average game length 3–6 days, real seer claims by day 3 in ≥ 60% of games, wolves never publicly accuse their partner while partner `publicSuspicion < 8`. Tune constants (not code) if out of bounds; record final rates in the PR.
8. Transcript readability review: generate 3 seeded game transcripts via `speechToText`, commit as fixtures, and have them reviewed in the PR for obvious nonsense (e.g., accusing the dead, voting confirmed-innocent without cause).

## Acceptance Criteria

- [ ] Weight-table unit tests: each §15.6 row exercised by a crafted event sequence with exact S deltas, for private S and for publicSuspicion (neutral), incl. persona-threshold behavior differences.
- [ ] Policy unit tests: seer claim triggers (all 3 + cautious shift), wolf deflection & bus threshold (crafted publicSuspicion 7 vs 8 fixtures), follower vote-mirroring, kill priority (a) > (b) > (c) with tie cases.
- [ ] Determinism: 20 seeded NPC-only games reproduce identical transcripts across two runs and across OSes (CI matrix).
- [ ] Calibration bounds met and reported (village 35–65%, length 3–6 days, seer-claim ≥ 60%); the 3 seeded readability transcripts are committed as fixtures and linked in the PR with reviewer notes.
- [ ] Structured-acts-only guarantee (ADR-005): the NPC-facing log entry type omits `flavor` at the type level, and a runtime assert strips it at the tracker boundary — a game where the agent's flavor text screams "Casey is the wolf" while its acts say nothing must produce byte-identical NPC decisions to the same game with empty flavor (paired-seed test).
- [ ] Knowledge hygiene: villager-persona NPC code paths receive only `viewFor(self)` + public log; wolf/seer extras injected explicitly (type-level test).

## Validation

- `test/unit/games/werewolf-npc.test.ts` + `tools/werewolf-sim.ts` calibration script (results in PR).

## Dependencies

- 04, 33, 34.

## Non-goals

- LLM NPCs (v2, ADR-002), plugin wiring (36), flavor-text generation for NPCs (they speak acts only in v1).

## Design References

- DESIGN.md §15.4–§15.7, §21 (U4), §18.2 (T3); ADR-002, ADR-005
