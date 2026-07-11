# Title

Bundled tutorial escape scenario: `intro-cell`

## Summary

Author and commit `content/escape-scenarios/intro-cell.json` per DESIGN.md §14.7: a 2-room, ~6-entity tutorial (par 15) that teaches every verb — examine, take, use, open, move, inventory, hint — with a single win ending.

## Context

The tutorial is most agents' first mcparcade experience after `list_games`; it must be self-explanatory, forgiving, and complete a clean teaching arc for the verb set. Pure content work against the schema (28) and engine (29).

## Scope

- The scenario file (validated), including hint tiers 1–2 and one `once` interaction.
- A golden harness transcript test playing it to the win.
- Design sketch (in the scenario file's `brief` and a short comment block in the test): room map, item chain, par math.

## Detailed Requirements

1. Structure: `cell` (start) and `corridor`; win = unlock the corridor's end door and `end_game`.
2. Teaching beats, in order (exact ids in parentheses): `look` (room lists `cot`, `loose-brick`), `examine cot` → `reveal_item {item:"bent-spoon", room:"cell"}`, `take bent-spoon`, `use bent-spoon` on `loose-brick` (requires `has_item bent-spoon`) → `reveal_item {item:"iron-key"}` + `set_flag {flag:"brick-moved", value:true}`, `take iron-key`, `move north` blocked (exit cell→corridor has `lockedBy:"cell-door-locked"`, lockedText points at the door), `use iron-key` on `cell-door` → `unlock_exit {room:"cell", dir:"north"}` with `once:true`, `move north`, `open exit-door` → `end_game {ending:"escaped"}`. Declared flags: `brick-moved`, `cell-door-locked`. The final `exit-door` needs no key: its `open` interaction requires only `in_room corridor` (the tutorial's last beat stays friction-free).
3. Every entity has a description that nudges the next step without spelling it out; `brief` explains the verb set explicitly (it's the tutorial).
4. Hints: tier 1 after turn 8 ("the cot hides something", penalty 25), tier 2 after turn 12 (explicit next step, penalty 50).
5. Par: 15 turns = the minimal-verb solve computed from the golden transcript (recount after authoring; par must equal the optimal transcript's turn count so a perfect run scores 1000).
6. Language: English, plain, ≤ 2 KiB per text field; no meta-instructions to the AI (T8 hygiene: content never addresses "you, the assistant" — it addresses the player character).
7. File passes `validate-scenario` with zero warnings (fix orphans rather than shipping warnings).

## Acceptance Criteria

- [ ] `mcparcade validate-scenario content/escape-scenarios/intro-cell.json` → exit 0, zero errors, zero warnings.
- [ ] Two golden transcripts: (a) optimal solve, **no hints** → score exactly 1000, `ending: "escaped"`, covering examine/take/use/open/move/inventory; (b) a fixed scripted path that waits past turn 8, takes hint tier 1, and finishes at a scripted turn count → asserted score computed as `1000 − 25 − 10·max(0, turns − 15)` with the transcript's exact turn count hard-coded.
- [ ] Every verb (examine/take/use/open/move/inventory/hint) appears at least once across the two transcripts (hint only in (b) — a hint-free optimal run is what makes score 1000 possible per §14.6).
- [ ] The tutorial completes in ≤ 20 accepted actions on the optimal path.
- [ ] CI bank test (from issue 29's suite) loads and validates all committed scenarios including this one.

## Validation

- `test/integration/games/escape-intro.golden.test.ts`; manual: one human read-through of all text for tone/clarity.

## Dependencies

- 28, 29.

## Non-goals

- The main scenario (31), Japanese localization (v2), puzzle difficulty.

## Design References

- DESIGN.md §14.7, §14.2, §14.6, §18.2 (T8)
