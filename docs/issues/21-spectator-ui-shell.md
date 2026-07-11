# Title

Spectator SPA shell: session list, generic live session view, event log

## Summary

Build the bundled single-page app (`src/spectator/assets/`) per DESIGN.md §11.6: vanilla TypeScript, hash routing, token bootstrap + address-bar scrub, live session list, and a generic session view (monospace render + players/score panels + public event log) that works for every game with zero per-game code.

## Context

The generic view is the universal fallback — per-game pretty renderers (23/27/37) plug into it later. Strict CSP (`default-src 'self'`, no external anything) and textContent-only DOM building are hard security requirements (T6).

## Scope

- Build integration: SPA TS compiled by tsup as a second entry, emitted into the asset manifest (issue 19) — no external fonts/CSS/JS; one hand-written CSS file (dark theme, monospace board area, light/dark via `prefers-color-scheme`).
- Bootstrap: read `?token=`, store in memory (not localStorage), `history.replaceState` to scrub URL (§11.2); all fetches use `X-Mcparcade-Token` header.
- Routes: `#/` sessions list (active + recent finished, live-updating), `#/s/:id` session view. (`#/r/:id` replay view is issue 22; nav stub only.)
- Session view: `<pre>` render, panels (players, status/turn, score/outcome), public event log (last 100, autoscroll), SSE-driven refetch, per-game renderer registry `window.__renderers` keyed by `web.kind` with graceful fallback to the `<pre>`.
- Empty/error states: no sessions yet; 401 (token missing/rotated after server restart) → full-page "re-open from the URL in your terminal" message.

## Detailed Requirements

1. No framework, no runtime deps; DOM built via `document.createElement`/`textContent`; the **only** `innerHTML` allowed is with compile-time constant strings (grep-tested).
2. `esc()` helper exists but rendering must not need it for correctness (textContent-only rule); event-log summaries and flavor text are always text nodes.
3. SSE client: `EventSource` with the token as a query param (the §11.2-sanctioned SSE mechanism — EventSource cannot set headers), auto-reconnect default, list/detail refetch on relevant ids only.
4. Renderer plug-in contract (consumed by 23/27/37), normative: the shell module exposes `registerRenderer(kind: string, fn: (container: HTMLElement, web: object, ctx: {gameId: string, status: string, finished: boolean}) => void)`; the shell calls `fn` on every update with the same container element and the fresh `web` model — renderers must be idempotent full-redraws that own the container's children; any exception thrown by a renderer is caught by the shell, logged to console, and replaced with the `<pre>` fallback for that update.
5. Accessibility floor: semantic headings, focus-visible, `aria-live="polite"` on the event log; keyboard-reachable session list.
6. Payload discipline: list view renders from `/api/sessions` summaries only; detail fetch only for the open session.
7. Title bar shows the game title (from the detail response's `meta.gameTitle` — issue 19's shape), sessionId (short form), status badge, turn counter; finished sessions show outcome banner.

## Acceptance Criteria

- [ ] `npm run build` emits assets into the manifest; the bundle contains no external URL literals (grep for `https?://` across emitted assets); rendering the shell against fixture API responses in jsdom produces zero `console.error` calls (spy).
- [ ] Token scrub: after load, `location.href` contains no token (jsdom or headless assertion).
- [ ] With two stub sessions, list shows both; driving one via the harness updates its row (turn/status) without reload (SSE integration test using a real browser-less EventSource polyfill in vitest… if impractical, split: DOM logic unit-tested in jsdom with a mocked EventSource, and the SSE wire covered by issue 20's tests — document which path was taken in the PR).
- [ ] Session view of a finished stub game shows render text, outcome banner, ≤ 100 log lines.
- [ ] Unknown `web.kind` falls back to `<pre>` cleanly.
- [ ] Grep tests: no `innerHTML` with non-constant input; no `localStorage`; no external URL literals in assets.

## Validation

- `test/unit/spectator/ui-shell.test.ts` (jsdom for DOM logic, grep-guards) + manual smoke checklist in the PR (open UI, watch a live maze game).

## Dependencies

- 19, 20.

## Non-goals

- Replay viewer (22), per-game renderers (23/27/37), acting from the browser (ADR-006), i18n.

## Design References

- DESIGN.md §11.2, §11.6, §18.2 (T6); ADR-006
