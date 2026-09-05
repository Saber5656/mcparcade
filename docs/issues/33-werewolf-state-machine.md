# Title

Werewolf phase state machine, roles, and win conditions

## Summary

Implement `src/games/werewolf/machine.ts`: the pure phase/role/win-condition core per DESIGN.md §15.1–§15.3, §15.5 — setup, night/day cycle, discussion rounds, simultaneous votes with runoff, executions, reveals, and win checks. No NPC decision logic (issue 35) and no plugin wiring (issue 36).

## Context

Werewolf is the most stateful game; isolating the deterministic machine from NPC "brains" keeps both testable. The machine consumes *resolved inputs* (everyone's night choices / votes / utterances) and advances phases; who chooses what is upstream.

## Scope

- Types: `Role = villager|werewolf|seer`, `Phase` per §15.2 (`night(k)`, `day(k).announce|discussion(round r, seat s)|vote|runoff|execution`, `finished`), `WolfState` (players, seats, roles, alive, phase, publicLog, privateFacts, votes, nightChoices, dayCount).
- `setup(playerCount, agentRole, rng)`: role assignment (§15.1 role sets), seat shuffle, name assignment.
- Transition functions (pure): `submitNight(state, playerId, choice)`, `resolveNight(state)`, `submitUtterance(state, playerId, utterance)`, `advanceSpeaker(state)`, `submitVote(state, playerId, target)`, `resolveVote(state, rng)` (plurality → runoff among tied → rng pick), `winCheck(state)`.
- Rule details: night-1 no kill (§15.2), seer divines night 1, execution reveal (role public on death), lead-wolf kill choice (lowest seat alive wolf), self/dead-target input rejection.

## Detailed Requirements

1. Phases are explicit data (`{kind:"discussion", day:2, round:1, seat:4}`), never inferred; every transition validates the input belongs to the current phase and player (wrong-phase input → typed rejection the plugin maps to `E_NOT_YOUR_TURN`).
2. Vote resolution (§15.5 normative): plurality of alive players; tie → one runoff among tied candidates (everyone revotes, only tied are votable); tie again → `rng("game").pick` among tied. Executed player's role is appended to publicLog; deaths from night kills reveal role at dawn announce.
3. Win check after every death event: wolves = 0 → village; wolves ≥ non-wolves alive → wolves. Agent's outcome = its faction's result (§15.8).
4. `privateFacts` per player: role; wolves → wolf list + nightly chosen target; seer → divination results. Machine exposes `viewFor(playerId)` and `viewPublic()` — the *only* accessors the plugin may use (leak-safety by construction).
5. Divination result of the executed-that-night edge: seer divines a player killed the same night → result still delivered (standard: actions resolve simultaneously; document).
6. Utterances are stored verbatim in publicLog as `{seat, playerId, acts, flavor}` (schema from issue 34); the machine validates acts structurally but attaches no meaning (NPC interpretation is issue 35).
7. Determinism: all rng via injected streams; seat/name/role assignment reproducible from seed (golden setup test).
8. Turn budget: `maxTurns: 200` counts global turns exactly like every game (§15.2/§7.2); with ≤ 9 players and bounded rounds a full game stays well under 150 global turns — assert in the property simulation.
9. Setup determinism (normative algorithm): with the `init` stream — (a) `seatOrder = rng.shuffle(playerIds)` (p1 = agent, p2.. = NPCs); (b) `roles = rng.shuffle(roleSetArray)` assigned to seats ascending (seat 0 gets roles[0], …); (c) NPC display names assigned from the fixed name list in player-index order (p2 → "Ash", p3 → "Blake", …). With `agentRole` fixed (practice), re-deal only the agent's card: swap the agent's dealt role with the nearest seat holding the requested role (deterministic: lowest seat index).
10. Free-text `flavor` arrives ≤ 1 KiB (runtime cap, §7.4 — enforced upstream in issue 09); the machine stores it verbatim in the public log and never interprets it; rendering/log hygiene are the spectator's (T6) and logger's (T13) jobs.

## Acceptance Criteria

- [ ] Golden setup: seed `0000000000000001`, 7p, random role → snapshot seats/names/roles.
- [ ] Full scripted 7p game (all inputs scripted, no NPC logic) reaching village win via 2 executions + 1 divination-guided vote; and a wolf-win script (wolves reach parity). Phase trace snapshots match §15.2 order exactly, including night-1 no-kill.
- [ ] Vote tie → runoff → tie → deterministic rng pick (fixture with forced ties both rounds).
- [ ] Reveal rules: night victim's role public at announce; executed role public at execution; no other role in `viewPublic()` (string-level assert on serialized view).
- [ ] `viewFor(villager)` of a mid-game state contains zero occurrences of other players' roles or wolf-list markers (leak fixture reused by issues 36/39).
- [ ] Wrong-phase/self-vote/dead-target inputs → typed rejections; state unchanged.

## Validation

- `test/unit/games/werewolf-machine.test.ts` — scripted-game suites + property test: 200 random-input games (seeded) always terminate ≤ 60 agent-visible steps with a valid win result.

## Dependencies

- 03, 04, 34 (utterances are validated structurally with issue 34's schemas before storage).

## Non-goals

- NPC decisions (35), speech-act semantics beyond structural validation (34/35), plugin/runtime wiring (36), extra roles (medium/guard — v2).

## Design References

- DESIGN.md §15.1–§15.5, §15.7–§15.8, §7.4 (flavor cap), §18.2 (T3)
