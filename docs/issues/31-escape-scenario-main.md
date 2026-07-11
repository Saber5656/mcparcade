# Title

Bundled main escape scenario: `locked-lab`

## Summary

Author and commit `content/escape-scenarios/locked-lab.json` per DESIGN.md §14.7: the flagship ~9-room, ~30-entity scenario (par 60) with a multi-step dependency chain, one red herring, three hint tiers, and two endings (win + one lose trap).

## Context

This is the piece of content the escape category is judged by. It exercises the full schema: multi-room progression, flag chains, entity states, `once`, lose endings, and layered hints.

## Scope

- The scenario file; a room/dependency map as a Mermaid diagram committed in `docs/research/locked-lab-map.md` (design artifact, spoiler-marked).
- Two golden transcripts: optimal win; lose-ending path.
- Par calibration identical to issue 30's method.

## Detailed Requirements

1. Setting: a research lab after hours. Rooms (9): lobby, hallway, lab-a, lab-b, server-room, office, break-room, storage, loading-dock (exit). Win: restore power → find keycode fragments (2) → open the office safe → keycard → loading-dock door → `end_game(escaped)`.
2. Dependency chain ≥ 6 steps deep with two independent mid-chains (fragments discoverable in either order — test both orders in unit form, golden covers one).
3. Red herring: a locked cabinet whose key exists but contains only flavor items (never blocks progress; validator warnings avoided by giving it a harmless interaction).
4. Lose ending: triggering the server-room's alarm (a clearly telegraphed risky interaction with `requires` allowing it only before power is rerouted) → `end_game(caught)` kind lose. It must be avoidable with attentive play and never reachable by accident in the optimal path.
5. Hints: 3 tiers (after turns 20/35/50; penalties 25/50/100), each tier phase-aware wording (written to help wherever the player is stuck without spoiling the endgame).
6. Text budget: total file ≤ 128 KiB (half the cap); every description ≤ 1 KiB; entity names unique enough for name-resolution (no two "key"s — use "brass key", "keycard").
7. T8 hygiene (self-contained, per DESIGN.md §18.2 T8 / §14.8): all prose addresses the player character, never the assistant; no meta-instructions of any kind (no "ignore previous…", "you are an AI…", "system prompt", imperative instructions aimed at a model — the issue-39 tripwire denylist must not match); the `author` field is inert metadata; also no real-world brand names, no license-encumbered text.
8. Zero validator errors/warnings.

## Acceptance Criteria

- [ ] `validate-scenario` → exit 0, zero warnings.
- [ ] Optimal golden transcript → win, score 1000 (par equals optimal turns), both fragment orders solvable (order-B verified via a second shorter scripted segment or unit-level interaction assertions).
- [ ] Lose golden transcript → `ending: "caught"`, score 0, and the alarm interaction is *not* triggerable after power reroute (fixture assert).
- [ ] Red-herring chain completable and score-neutral.
- [ ] Hint tiers gate at 20/35/50 and stack penalties correctly in a scripted hint-heavy run.
- [ ] Play-through by a fresh reader (human, or a non-Claude agent via `codex exec` playtest) recorded in the PR as evidence: the action transcript (or tool log), completion status, turns used, hints used. "Without external help" = the player never reads the scenario JSON or the map doc; hints via the in-game `hint` action are allowed and reported. If the playtester exceeds par × 2 turns at any single puzzle, revise that puzzle's descriptions and re-test.

## Validation

- `test/integration/games/escape-lab.golden.test.ts` (+ the order-B unit asserts); playtest note in the PR.

## Dependencies

- 28, 29.

## Non-goals

- More scenarios/endings, timed mechanics, NPCs.

## Design References

- DESIGN.md §14.7, §14.2–§14.6, §18.2 (T8)
