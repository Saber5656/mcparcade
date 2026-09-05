# Title

Shogi board renderer in the spectator UI

## Summary

Add a `kind: "shogi"` renderer to the SPA registry (issue 21) drawing the 9×9 board with kanji pieces, hands, last-move highlight, and check indicator from issue 26's `renderWeb` model.

## Context

Shogi is the flagship "watch your agent think" game; the text `<pre>` works but a real board makes spectating and replay review dramatically better. Pure SPA work; no server changes.

## Scope

- Board: CSS grid 9×9 with rank/file labels (１-９, 一-九), pieces as text glyphs (`歩`, `と`, `馬`…), white pieces rotated 180° via CSS transform, last-move cell highlighted, king cell pulsing subtly when `check`.
- Hands: two tray rows (`☗` / `☖`) with piece×count chips.
- Move list panel: from the session's public event log (already in the shell) — ensure kanji summaries align (no new data needed).
- Works in live view and replay frames identically.

## Detailed Requirements

1. Model consumed as-is from issue 26 (normative there): `{kind:"shogi", board: [{file, rank, piece, owner}], handsBlack, handsWhite, lastMove: {file, rank}|null, check, checkedKing: {file, rank}|null}`. Piece codes (complete domain): `P L N S G B R K +P +L +N +S +B +R`. Glyph map piece-code→kanji lives here (single source `SHOGI_GLYPHS`: 歩香桂銀金角飛玉と杏圭全馬龍). Coordinate mapping (normative): CSS-grid column = `10 − file` (file 9 renders leftmost), row = `rank` (rank 1 at top) — black at bottom.
2. Hands trays: render only piece types with count ≥ 1, in the fixed order `R B G S N L P`, as `<kanji>×<count>` chips; negative/non-integer counts render as `?` (defensive).
3. textContent-only DOM (issue 21 rules); board width `min(80vmin, 560px)`; the stylesheet must contain a ≤ 360 px media rule shrinking board font so 9 columns fit without horizontal overflow (static CSS assertion in tests + manual check).
4. Missing/malformed fields degrade: unknown piece code renders `?` cell; absent hands render empty trays; never throw. White pieces rotate 180° via CSS transform — this is the web equivalent of the text render's `v` prefix (§13.6 governs the text surface only; no conflict).
5. Move list: no new data — the shell's `panels.log` already carries kanji move summaries from game events (§13.6); this issue asserts one fixture log line matches the `☗７六歩` format to pin alignment.
6. Orientation: always black-at-bottom in v1 (the agent's side is shown in the players panel).
7. No animations beyond the check pulse (on the `checkedKing` cell) and a 150 ms last-move fade-in (CSS only).

## Acceptance Criteria

- [ ] jsdom fixture render (midgame model captured from issue 26 tests): correct glyph at 5 spot-checked (file, rank) coordinates incl. one promoted piece, white pieces carry the rotated class, last-move cell highlighted, hands chips show `歩×3`-style counts in the fixed order.
- [ ] `checkedKing` present → indicator class on exactly that cell; `check: false` → no indicator anywhere.
- [ ] Malformed model degrades per requirement 4 (three fixtures: unknown piece code, missing hands, missing board).
- [ ] Panels log line format matches `☗７六歩` on the fixture (requirement 5).
- [ ] Replay stepping re-renders without leaking nodes (container child count stable across 50 re-renders).
- [ ] Stylesheet contains the ≤ 360 px media rule (static assert).

## Validation

- `test/unit/spectator/shogi-renderer.test.ts` (jsdom fixtures) + manual smoke: start `shogi {difficulty:"easy", seed:"0000000000000003"}` via a real client, open the spectator, verify board/hands/last-move/check rendering against the text render at three points in the game (checklist in the PR).

## Dependencies

- 21, 26.

## Non-goals

- Click-to-inspect squares, engine eval bars, orientation flip, sound.

## Design References

- DESIGN.md §11.6, §11.7, §13.6
