# Title

Maze game plugin: seeded generator, fog mode, path action, optimality scoring

## Summary

Implement `src/games/maze/` as the first real `GameDefinition` (DESIGN.md §12.1): recursive-backtracker maze, `move`/`path` actions, fog-of-war option, BFS-optimal scoring, text render, and grid web-render model.

## Context

Wave 1 exists to prove the whole pipeline with simple engines. Maze exercises: seeded generation, multi-step actions, per-cell rendering, score-game finalization, and the `web` grid model consumed later by issue 23.

## Scope

- `GameDefinition` with `id: "maze"`, `category: "puzzle"`, `scoring: "score"`, `maxTurns: 2000`, `version: "1.0.0"`.
- Options schema: `{size?: 9|15|21 = 15, fog?: boolean = false, seed?}`.
- Actions: `{"type":"move","dir":"up|down|left|right"}` · `{"type":"path","dirs":[dir,...] (1..64)}` · runtime resign.
- Generator, BFS optimal-length calc at init, fog visibility set, renders.

## Detailed Requirements

1. Generation (rng stream `init`): grid of odd `size`; cells at odd coords are rooms; recursive backtracker with neighbor order chosen via `rng.shuffle(["up","right","down","left"])` per step (documented so replays are stable). Entrance `S` = (1,1), exit `E` = (size−2, size−2).
2. `path` validates the full step sequence against the maze before moving: on the first illegal step the whole action is **rejected** with hint "step k (dir) hits a wall" (k is 1-based) and no movement (keeps determinism simple and teaches precise planning).
3. Fog mode: `visited ∪ currently-visible` = cells within Chebyshev distance 1 plus straight-line sight down corridors until a wall; unseen renders as space. State exposes `seen` count.
4. Structured state (normative): `{size, fog, grid: string[]}` where `grid` is `size` strings of `size` glyphs (viewer-adjusted in fog), plus `{pos: {r, c}, exit: {r, c}, steps, optimal, seen}` — `r`/`c` are 0-based from top-left. Render per §12.1 glyphs (`#`, `·`, `@`, `S`, `E`); glyph precedence: `@` overrides `S`/`E` on shared cells; ≤ 16 KiB even at 21×21 (trivially true).
5. `legalActions`: list mode with the ≤ 4 legal `move` dirs. The `path` action is intentionally NOT enumerated in `legalActions` (it would explode combinatorially) — it is taught via `describe_game`/rulesDoc and the schema-mode summary line included in the rejection hints.
6. Terminal: player on `E` → win; `score = round(1000 * optimal / max(steps, optimal))`; `summaryExtras: {size, optimality: round(100*optimal/steps)}`. Resign → loss score 0 (runtime).
7. Events: each accepted action emits one `move` event `{summary: "moved up ×3"}`.
8. `renderWeb`: `{kind:"grid", rows, cols, cells:[{r,c,glyph,cls:"wall|floor|you|start|exit|fogged"}]}` (public view = full map even in fog — the *spectator* may see all; the agent's fog is in `observe`).

## Acceptance Criteria

- [ ] Fixed seed `0000000000000001`, size 15: snapshot the full render (golden); regenerate twice → identical.
- [ ] Generated maze is *perfect* (property test over 50 seeds: exactly one path between any two rooms — assert via spanning-tree edge count = rooms−1, and S→E reachable).
- [ ] `path` rejection leaves position unchanged (byte-equal state).
- [ ] Fog: cells beyond sight absent from render; revisited cells persist.
- [ ] Golden transcripts (harness): scripted optimal solve on the fixed seed → win, score 1000, optimality 100; a second transcript on the same seed that prepends 6 extra legal moves (recorded once when authoring the fixture) asserts `score === round(1000 * optimal / steps)` computed from the observed `optimal` and `steps` (objective without hard-coding the maze).
- [ ] Determinism: replayToTurn over the golden game reproduces final state.
- [ ] Profile after two harness games records `bestScore` and `lastExtras: {size, optimality}` per §10.3.

## Validation

- `test/unit/games/maze.test.ts` + `test/integration/games/maze.golden.test.ts` (harness `playTranscript`).

## Dependencies

- 04, 09, 14.

## Non-goals

- Spectator SPA rendering of the grid model (23). Difficulty beyond size/fog.

## Design References

- DESIGN.md §12.1, §6, §10.3, §11.6
