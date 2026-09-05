# Title

Spectator grid renderers: maze, 2048, sudoku pretty views

## Summary

Implement the `kind: "grid"` renderer in the SPA renderer registry (issue 21), with per-game styling for maze cells, 2048 tiles (value-scaled colors), and sudoku (givens vs entries, 3×3 box borders).

## Context

Wave-1 games emit `renderWeb` grid models (issues 15–17). This issue is pure SPA presentation — additive polish over the always-working `<pre>` fallback; it must never break when models evolve (defensive rendering).

## Scope

- `grid` renderer: CSS-grid layout from `{rows, cols, cells}`; cell content via `textContent`; per-game classes driven by cell fields (`cls` for maze, `value` for 2048, `given` for sudoku).
- Styling: maze (wall/floor/you/exit contrast, fog dim), 2048 (log2-value hue ramp, empty cells subtle), sudoku (box borders every 3, givens bold, entries normal).
- Live + replay reuse (registry is shared; no special-casing).

## Detailed Requirements

1. One renderer keyed `grid`; game specialization by `ctx.gameId` → class prefix using the gameId verbatim (`grid--maze`, `grid--g2048`, `grid--sudoku`). Consumed cell model (normative union of what issues 15–17 emit): `{kind:"grid", rows, cols, cells: [{r, c, glyph?, cls?, value?, given?}]}` with 0-based `r`/`c`; cell text = `glyph` if present else `String(value)` if present else empty. Unknown/missing cell fields render as plain text cells (never throw).
2. Cells are `div`s in a CSS grid; container width `min(70vmin, 640px)`, cell size driven by a `--grid-cols` CSS custom property set from `cols` (so a 21×21 maze yields ≥ 12 px cells inside 640 px — legibility floor).
3. 2048 tile colors: computed class `v2…v2048plus` (log2 bucket), defined in the single CSS file; no inline style computation from data beyond the class name mapping (CSP/injection hygiene). Sudoku: `given` cells render bold, player entries regular weight — this is a spectator-side affordance; the agent-facing text render stays undifferentiated per DESIGN.md §12.3 (no conflict: different surfaces).
4. Re-render on update replaces container contents wholesale (simplicity over diffing; grids are ≤ 441 cells).
5. Zero changes to games or server; if a model field the CSS expects is absent, degrade to neutral styling (test this).

## Acceptance Criteria

- [ ] jsdom render of fixture models captured from the games' `renderWeb` outputs: maze 15×15 on issue 15's golden seed — `you` class at the entrance cell (r=1,c=1) in the initial model and `exit` class at (r=13,c=13); 2048 — a 128 tile carries `v128`; sudoku — `given` cells carry the given class, box-border classes present at row/col indices 3 and 6.
- [ ] Malformed model (missing `cells`) → fallback message inside the container, no exception.
- [ ] Idempotent re-render: 50 consecutive renders of alternating models leave exactly one grid in the container (no node leaks), which also covers live/replay reuse (both call the same registered renderer per issue 21's contract).
- [ ] Grep-guards from issue 21 still pass (textContent-only).

## Validation

- `test/unit/spectator/grid-renderer.test.ts` (jsdom, fixture models from the three games' `renderWeb` outputs captured as fixtures); manual smoke on a live 2048 game.

## Dependencies

- 15, 16, 17, 21.

## Non-goals

- Shogi/werewolf renderers (27/37), animations, themes beyond light/dark.

## Design References

- DESIGN.md §11.6, §12.1–§12.3 (`renderWeb` models)
