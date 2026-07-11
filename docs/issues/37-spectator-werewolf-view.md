# Title

Werewolf spectator view: public log, phase timeline, post-game reveal

## Summary

Add the `kind: "chatlog"` renderer to the SPA (issue 21 registry) for werewolf per DESIGN.md §15.8/§11.6: live public-only discussion/vote view with a phase timeline, and the full post-game reveal (roles, night choices, suspicion trajectories) once a game finishes.

## Context

Werewolf spectating is the product's best demo (humans love watching their agent bluff). The hard rule: while active, the browser must see exactly what a dead-outsider would — the server already enforces public-only (issues 19/36); this issue renders it and the finished-game reveal.

## Scope

- Live view: phase banner (day/night, round, whose seat speaks), alive/dead roster (dead show revealed roles), speech log (acts as chips + flavor as quoted text), vote tally table per day, night announces.
- Reveal view (finished sessions/replays), sourced from the `reveal` field of `GET /api/sessions/:id` / `GET /api/replays/:id` (shapes fixed by issues 19/22/36): role badges on all players (`reveal.roles`), night-kill choices per night (`reveal.nightKills`), seer divination log (`reveal.divinations`), and one sparkline per player from `reveal.trajectories[name]` — an array of that player's shared publicSuspicion value at each dawn — rendered as tiny inline SVG `<polyline>`s built from the numbers (no external lib).
- Replay integration (§11.7): frames are always the public projection; the reveal panel unlocks at the final frame, with a "spoilers" toggle to reveal early — default off.

## Detailed Requirements

1. Acts render as typed chips (`claims seer`, `divined Drew: WOLF`, `accuses Casey (contradiction)`, `defends Drew`, `will vote Casey`, `passes`) — mapping table in one place; unknown act kinds render as plain text (forward-compat).
2. Every string this view injects — player names, act chips, targets, flavor, notices, reveal fields — is inserted via `textContent`/the shell's `esc()` helper, never markup (T6/§11.6; flavor and names are agent-authored free text).
3. Vote tables: one per day incl. runoffs, executed player highlighted.
4. Sparklines: pure SVG `<polyline>` from integer arrays; axis-free, tooltip shows day:value via `<title>`; degrade to a numeric row if trajectories absent.
5. Spoiler toggle state is per-page-load (memory only), default hidden until final frame.
6. Zero server changes; consumes `publicView`/`revealPayload` shapes from issues 19/36 (if a field is missing, degrade gracefully — never throw).

## Acceptance Criteria

- [ ] jsdom fixtures: live mid-game model renders phase banner, chips for all 6 act kinds, dead player with revealed role, vote table; no role-badge element exists for any alive player, and fixture sentinel role tokens (unique strings standing in for `revealedRole` values of alive players) appear nowhere in the DOM (speech-act chips like "claims seer" are public info and legitimately present).
- [ ] Injection fixture: a player named `<img src=x onerror=…>` with flavor `<script>` renders as inert text (no element creation — query assert).
- [ ] Finished-game model renders reveal badges, night choices, divination log, sparklines (polyline point count = days).
- [ ] Replay: spoiler toggle hides/shows reveal panel; final frame auto-unlocks.
- [ ] Malformed/missing `revealPayload` degrades to public view with a notice.
- [ ] Grep-guards (21) still green.

## Validation

- `test/unit/spectator/werewolf-view.test.ts` (jsdom fixtures captured from issue 36's golden games); manual smoke with explicit checks recorded in the PR: (1) during a live 7p game, no roles visible for alive players at day 2; (2) after the game ends, reveal panel shows roles/night-kills/divinations/sparklines; (3) in the replay, frames stay public and the spoiler toggle gates the reveal panel until the final frame.

## Dependencies

- 21, 36.

## Non-goals

- Live role-peeking / omniscient mode (v2, ADR-006), sound/animations, editing.

## Design References

- DESIGN.md §15.8, §11.6, §18.2 (T3, T6); ADR-006
