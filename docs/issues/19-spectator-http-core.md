# Title

Spectator HTTP core: loopback bind, token auth, security headers, static assets, read-only JSON API

## Summary

Implement `src/spectator/server.ts` + `api.ts` per DESIGN.md §11.1–§11.4 and ADR-006: the token-gated loopback HTTP server with hardened headers, bundled-asset serving, and the read-only session/replay JSON API (SSE is issue 20; SPA content is issue 21).

## Context

This is the product's only always-on network surface, so its security posture (T4/T5/T6/T10 in §18.2) is the core of the issue, not an add-on. Built on `node:http` — no framework dependency.

## Scope

- Server lifecycle: `startSpectator(config, deps) → {url, close}` wired into the CLI seam from issue 13; bind `127.0.0.1`, port from `--spectator-port` (default 0 → ephemeral, report actual).
- Token: `crypto.randomBytes(16).toString("hex")` per process; auth check helper (query `token` or `Authorization: Bearer` or `X-Mcparcade-Token`) with `timingSafeEqual`.
- Request pipeline (normative order): Host allowlist (403) → Origin check (403) → method check (non-GET → 405, all routes are GET-only) → auth (401) → route (404). All error responses are JSON `{"error": "<snake_case_reason>"}`. Auth exemption (§11.4): `/assets/*` skips the auth step only — `<script>`/`<link>` requests cannot carry the token after the URL scrub; assets are public build artifacts with no user data. Request logging records the path **without query string** and never logs auth headers (§11.3, T10).
- Routes (§11.4): `/`, `/assets/*`, `/api/meta`, `/api/sessions`, `/api/sessions/:id`, `/api/replays`, `/api/replays/:id` (frame route is issue 22; `/api/events` is issue 20 — return 501 stubs for both, registered now so the route table is complete).
- Response shapes (normative): `/api/sessions?status=active|finished|all` (default `active`) → `[{sessionId, gameId, gameTitle, status, turn, updatedAt, summary}]`; `/api/sessions/:id` → `{meta: {sessionId, gameId, gameTitle, status, turn, updatedAt, outcome}, publicView: {render, web, panels}, reveal: object|null}` (`reveal` is populated only for finished social-category games from the plugin's reveal payload — null until issue 36 supplies it); `/api/replays` → `[{replayId, gameId, gameTitle, outcome, endedAt}]`; `/api/replays/:id` is specified in issue 22.
- `publicView` projection: for a stored session, compute `{render, web, panels}` per §11.6 — `web` from `renderWeb(state,"public")` when defined (else `null`), `render` from `renderWeb`'s text sibling when provided else `observe(state, "p1", false).render` (`p1` is the single agent player — a v1 invariant; safe for non-hidden-info games only); hidden-info games **must** have `renderWeb` (assert at registry init: `category === "social"` ⇒ `renderWeb` present).
- Security headers on every response (§11.3 exact list); `Cache-Control: no-store` on `/api/*`, `immutable` on `/assets/*`.

## Detailed Requirements

1. Host check: exact-match against `127.0.0.1`, `localhost`, `[::1]` with optional `:port` — reject otherwise with 403 `{"error":"host_not_allowed"}` **before** auth (rebinding defense; §11.3).
2. Origin: if header present and not in the same allowlist (scheme http) → 403. No CORS headers ever.
3. Asset serving from an in-code manifest (path → {bytes|filePath, contentType}) generated at build time from `src/spectator/assets/`; no directory traversal possible because lookups are manifest-key-only (never fs paths from URLs).
4. `:id` params validated with `isValidId` before store access (§18.4).
5. While a `social`-category session is `active`, `/api/sessions/:id` serves the public view only and `reveal: null` (assert no role/private fields — coordinate the leak test with issue 39); after `finished`, it serves the final public view plus the `reveal` field passthrough (whatever the plugin stored; null until issue 36 exists — this issue implements the plumbing, not the payload).
6. Auth failures: 401 JSON, `WWW-Authenticate: Bearer`; never echo the presented token; constant-time compare against the real token.
7. Token never written to disk or logs; `{url}` returned to the CLI contains `?token=…` for the human (stderr print happens in issue 13's seam — the URL log line is the single sanctioned token emission). The SPA-side address-bar token scrub is issue 21's requirement, not this one's.
8. Body size: reject non-GET with 405 (read-only server, ADR-006).
9. `deps` are read-only stores + registry + a `bus` placeholder (issue 20); no runtime mutation APIs are reachable from this module (enforce by importing only store read functions).

## Acceptance Criteria

- [ ] Boots on ephemeral port; `GET /api/meta` with token → 200 `{version, games, watchMode}`; without → 401; wrong token → 401 (timing-safe compare used — code review + test that compare helper is `timingSafeEqual`-based); `GET /assets/<bundled>` without a token → 200 (auth exemption), with a bad Host → 403.
- [ ] Host/Origin matrix test: `Host: evil.com` → 403; `Origin: http://evil.com` → 403; `Origin: http://127.0.0.1:<port>` → 200; missing Origin → 200.
- [ ] Header assertions on `/` and `/api/sessions`: CSP exact string from §11.3, nosniff, no-referrer, no-store (api), no `Access-Control-*` anywhere.
- [ ] POST/PUT/DELETE to any route → 405 even without a token (method check precedes auth per the normative pipeline order).
- [ ] Token hygiene: with a sentinel token, serialized bodies of every API response and the captured stderr (except the single startup URL line) contain zero occurrences of the token.
- [ ] `/api/sessions/:id` with traversal-ish ids → 404 without store access (spy); with a finished stub-npc session → publicView JSON with render+panels.
- [ ] Active social-stub session (fixture with `renderWeb` public view) → response JSON contains no `role`/private markers (string assert on fixture markers).
- [ ] Server listens on 127.0.0.1 only (connect via the machine's LAN IP fails — assert via `server.address()` and a direct socket attempt to `0.0.0.0` route is skipped as untestable-in-CI; the address assertion suffices).

## Validation

- `test/unit/spectator/http-core.test.ts` using real HTTP requests against an ephemeral instance (node fetch), temp-dir stores.

## Dependencies

- 03, 06.

## Non-goals

- SSE/live updates (20), the SPA itself (21), replay frames (22), HTTP MCP transport (38).

## Design References

- DESIGN.md §11.1–§11.4, §11.6, §18.2 (T4, T5, T6, T10), §18.4; ADR-006
