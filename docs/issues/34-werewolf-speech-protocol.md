# Title

Werewolf speech-act protocol: schemas and public log model

## Summary

Implement `src/games/werewolf/speech.ts` per DESIGN.md §15.4 and ADR-005: the zod schemas for utterances and each speech-act kind, validation limits, and the public-log entry format shared by machine, NPCs, plugin, and spectator.

## Context

The speech-act vocabulary is versioned API (ADR-005): it is the only channel into NPC reasoning and the backbone of the spectator's discussion log. Small, strict, and well-documented beats expressive.

## Scope

- `zUtterance`: `{type:"utter", acts: Act[] (1..4), flavor?: string ≤ 1024 chars}`.
- `Act` discriminated union (exact §15.4 list): `claim_role{role}`, `report_divination{target, result}`, `accuse{target, reason}`, `defend{target}`, `declare_vote{target}`, `pass{}` — `reason` enum `contradiction|vote_pattern|claim_timing|defended_wolf|gut`; `result` enum `werewolf|not_werewolf`; `role` enum `villager|werewolf|seer`.
- Target semantics: player *name* (the agent-facing identifier; e.g. "Casey"). Validation is contextual via `makeUtteranceSchema(ctx)` with `ctx = {roster: [{name, alive}], speaker: name}` — targets must be alive roster names, and self-target rules use `ctx.speaker`.
- Log entry type `SpeechEntry {day, round, seat, playerId, name, acts, flavor}` + compact text formatter `speechToText(entry)` (used in event summaries and renders: `Casey: [claims seer] [divined Drew: werewolf] "trust me"`).
- Documentation block (exported string) describing each act's meaning for `describe_game` (issue 36 embeds it).

## Detailed Requirements

1. Validation rules (all normative): 1–4 acts; **each act kind appears at most once per utterance** (so no double `claim_role`, no two `accuse`s); `pass` must be the only act when present; every target must be an **alive** roster name; self-target (`target == ctx.speaker`) is invalid for `accuse`, `report_divination`, and `declare_vote`, and valid for `defend`; `claim_role` has no target. Each rule failure yields its own teachable message.
2. `flavor` validation is strict, not lossy: control characters other than `\n` → **reject** (teachable message; the agent resubmits); length ≤ 1 024 chars checked on the raw string; flavor is never parsed by game logic (restate ADR-005 in a doc comment).
3. Schema errors must be agent-teachable: each failure path yields a one-line message usable as a rejection hint ("accuse.target must be an alive player name; alive: Ash, Blake, …").
4. `speechToText` output is deterministic plain text (no markup — spectator renders it via `textContent`, §18.2 T6): the acts part is ≤ 200 chars; if flavor exists, append ` "<flavor truncated at 80 chars with …>"` — hard total ≤ 285 chars. Full flavor stays in the log entry.
5. Pure module: no imports beyond zod/core types.

## Acceptance Criteria

- [ ] Fixture table: ≥ 12 valid utterances (every act kind, combos, defend-self) and ≥ 12 invalid (5 acts, duplicate kind, dead target, self-accuse, self-vote-declare, pass+extra, oversized flavor, control-char flavor) with exact error-message snapshots.
- [ ] Roster injection: same utterance valid/invalid as roster changes (alive→dead).
- [ ] `speechToText` snapshots for 5 representative entries.
- [ ] Property: any valid utterance round-trips zod → JSON → zod identically.

## Validation

- `test/unit/games/werewolf-speech.test.ts`.

## Dependencies

- 03.

## Non-goals

- NPC interpretation/weights (35), machine storage (33), spectator rendering (37).

## Design References

- DESIGN.md §15.4, §7.4 (free-text cap), §18.2 (T6); ADR-005
