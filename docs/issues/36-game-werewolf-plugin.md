# Title

Werewolf game plugin: views, night/day input flow, NPC integration, reveal rules

## Summary

Implement `src/games/werewolf/index.ts` — the `GameDefinition` wiring machine (33), speech (34), and NPCs (35) into the runtime per DESIGN.md §15: options, per-role observations, the simultaneous-phase input model, rating hookup, and post-game reveal payload.

## Context

The trickiest plugin: hidden information (T3), simultaneous phases inside a sequential runtime loop (§7.2), and practice-mode rating rules. Everything upstream is pure; this issue is orchestration + views.

## Scope

- `GameDefinition` `id: "werewolf"`, `category: "social"`, `scoring: "winloss"`, `maxTurns: 200`, `version: "1.0.0"`.
- Options (§15.1): `{playerCount?: 5|7|9 = 7, agentRole?: "random"|"villager"|"werewolf"|"seer" = "random", seed?}`; non-random `agentRole` ⇒ outcome `rated: false` (practice).
- Action schema: `utter` (via `makeUtteranceSchema(roster)`), `vote {target}`, `divine {target}`, `kill {target}`, `night_pass {}` — validated against the current phase.
- `currentPlayer` mapping over machine phases (per the §6.1 contract — never null while active): during discussion → the speaking seat's player (NPC seats drive the runtime NPC loop via `npcAct`); simultaneous phases (night, vote) → `p1` while the agent's input is pending, and the plugin resolves all NPC inputs inside the same `act` step (§15.5), then advances phase.
- `observe` views built strictly from `viewFor(playerId)`/`viewPublic()` (issue 33's leak-safe accessors); `you.private` per §15.7; `expected` field tells the agent what input kind is due.
- `renderWeb(state, "public")` (normative shape, consumed by issue 37): `{kind:"chatlog", phase: {kind, day, round?, seat?}, alive: [{name, seat, alive, revealedRole: role|null}], entries: SpeechEntry[] (issue 34 shape, incl. flavor), votes: [{day, tally: [{voter, target}], executed: name|null, runoff: boolean}], nightAnnounces: [{day, victim: name|null}]}` — all strings are inert plain text; the spectator renders them via `textContent` (T6 consumer contract).
- `revealPayload` (stored at `stateSnapshot.reveal` on finish; surfaced by issues 19/22 as the `reveal` field): `{roles: {name: role}, nightKills: [{night, chosenBy, target}], divinations: [{night, seer, target, result}], trajectories: {name: number[]}}` where `trajectories[name][d]` = the shared `publicSuspicion` value of that player at dawn of day d+2 (snapshot taken at each announce — plumb from issue 35's shared tracker).
- Rating: single 1200 anchor (§10.2/§15.8); win = faction win regardless of survival.

## Detailed Requirements

1. Turn-order rule: NPC discussion turns flow through the standard runtime NPC loop (`npcAct` returns an `utter` action) — one NPC utterance per `game.act` call, keeping events granular and the 200-step guard away (7p × 2 rounds = ≤ 18 NPC utterances per day, fine).
2. Night flow when the agent has no night action (villager): the observation's `expected: "night_pass"`; the `night_pass` action triggers resolution. Agent-as-lead-wolf sees teammates and submits `kill`; agent-as-non-lead-wolf submits `night_pass` and receives the lead's choice as a private event.
3. Rejections: wrong-phase actions → `E_NOT_YOUR_TURN`-mapped hint ("it is the discussion phase; expected: utter"); invalid utterance → schema's teachable message (34); dead-target etc. from machine rejections.
4. `render` (text): phase banner, alive roster with seats, last N speech lines (`speechToText`), vote tallies when public, your role card + private facts, expected-input line. ≤ 16 KiB enforced by truncating speech history (keep last 40 entries; full log via `state`).
5. rulesDoc: role explanations, phase walkthrough, the speech-act vocabulary (embed 34's doc block), bluffing guidance (false claims are legal moves), rating note, practice-mode note.
6. Leak tests are part of THIS issue (beyond 33's): serialize every agent-facing artifact (observation JSON, render text, events) for a villager-agent mid-game fixture and assert zero occurrences of hidden markers (wolf names as wolves, `role":"werewolf"` for others, seer results not yours). Fixture markers designed adversarially (e.g. role strings embedded in names must not false-positive — use unique sentinel tokens in fixtures).
7. Reveal timing: `outcome` + final observation include the full reveal; the spectator reveal view is issue 37 but the data (`revealPayload`) ships here on the finished session's `stateSnapshot`.
8. `describe_game` example section: 3 full `act` payload examples (utter with claim+report, vote, night divine).

## Acceptance Criteria

- [ ] Golden transcripts (harness, fixed seed): (a) agent villager, village win via correct voting — asserts phase progression, expected-input hints, rating +Δ; (b) agent wolf, wolf win including a bluff measured concretely — the agent utters `claim_role seer` + a false `report_divination` against a villager, and on that same day ≥ 1 NPC's vote targets the framed villager (assert via the vote tally), game ends in a wolf win; also asserts teammate visibility in `you.private` and its absence from the public log; (c) agent seer practice mode (`agentRole:"seer"`) → `rated: false`, no rating change.
- [ ] Leak suite (requirement 6) green — reused by issue 39.
- [ ] Simultaneous-phase correctness: agent vote submitted → same `act` response contains all revealed votes + execution + (if game continues) night transition up to the agent's next input point.
- [ ] Wrong-phase and malformed utterances rejected with teachable hints; state unchanged.
- [ ] Agent-death path (§15.5b): in a scripted game where the agent dies night 2, the death-processing `act` call plays the remaining game to terminal via the NPC loop and returns the full public event stream plus the faction-based outcome in that single response; no further input is expected.
- [ ] `maxTurns` safety: property sim (200 seeded games through the full plugin) never exceeds 150 global turns.

## Validation

- `test/integration/games/werewolf.golden.test.ts` + leak suite + property sim.

## Dependencies

- 09, 14, 33, 34, 35.

## Non-goals

- Spectator view (37), extra roles, in-game agent-to-NPC private chats (no whisper mechanic in v1).

## Design References

- DESIGN.md §15 (all), §7.2, §10.2, §18.2 (T3)
