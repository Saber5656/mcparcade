# Title

Cross-cutting security regression suite mapped to the threat table

## Summary

Implement `test/security/` per DESIGN.md §19.4: one regression suite that pins every §18.2 threat (T1–T14) to at least one executable test, consolidating and extending the security assertions scattered across earlier issues.

## Context

Individual issues each carry their local security ACs; this suite is the durable, single place where the whole threat table stays enforced as the codebase evolves (and where a security reviewer looks first). It must run in normal CI (no special environment).

## Scope

- `test/security/threat-matrix.test.ts` (or one file per boundary) with a visible T-number ↔ test mapping (describe blocks named `T4 dns-rebinding`, etc.).
- New tests where earlier issues had none; imports/reuse where they did (leak fixtures from 36, traversal tables from 05, header matrix from 19).
- Grep-guards as tests: stdout writes in `serve` paths (T14), `path.join(dataDir` outside `paths.ts` (T1), `Math.random` in `src/` (determinism), `innerHTML` with non-constants in SPA (T6), no `node-fetch`/`http.request` client usage outside `spectator/`+`server/http` (§18.3 no-egress).
- A header comment block in the suite documenting how to run it and how to add a threat row; SECURITY.md (issue 41) links to this suite — no file dependency in either direction.

## Detailed Requirements

1. Every T1–T14 row appears at least once; the file starts with a table comment mapping T# → test names → design ref. A CI-checked meta-test asserts the mapping table lists all 14 (no silent drop when refactoring).
2. T1: one shared traversal/fuzz id table applied to **every** id-accepting surface (§19.4 "every id-taking tool/route"): MCP `observe`/`act` (`sessionId`), `get_replay` (`replayId`), `describe_game`/`start_game`/`list_sessions`/`get_record` (`gameId` strings — bounded, unknown-id rejection), and spectator `/api/sessions/:id`, `/api/replays/:id`, `/api/replays/:id/frame/:t`.
3. T2: flood test — 25 rapid `start_game` calls (cap 20 → `E_TOO_MANY_SESSIONS`), 1 000 rejected actions with deterministic boundedness assertions (the rejection-counter map stays ≤ its documented LRU bound; the sessions dir contains exactly 20 `sess_*.json` files; no session file grows from rejected actions), event-cap breach path.
4. T3: werewolf leak fixtures (36) + spectator active-session leak (19) rerun here from the shared fixture module.
5. T4/T5/T6: spectator matrix (19/21 reuse) + one novel: SSE endpoint with forged `Origin` and with token-in-query-only (allowed) vs no token (401).
6. T7: escape-invalid corpus (28) fed through the *custom loading* path (32) — server boots, skips all, plays a bundled game fine.
7. T8: assert bundled scenario texts contain none of a denylist of meta-instruction phrases (`"ignore previous"`, `"you are an AI"`, `"system prompt"` — crude but a tripwire for content regressions); documented as heuristic.
8. T9: corrupt-file zoo per location, each with its documented behavior: `sessions/sess_*.json` + `replays/rep_*.json` (truncated, binary junk, 1 MB deep-nested JSON, 8 MiB+ oversized) → quarantined on access, boot + list + play unaffected; `profile.json` corrupt → quarantined + fresh default (issue 07); `scenarios/*.json` invalid → skipped with warning (issue 32) — all through real boot, not unit mocks.
9. T10: with a sentinel spectator token, assert zero occurrences in: all API response bodies, all persisted data-dir files, and captured stderr *except* the single sanctioned startup URL line (§11.2); request logs must show paths without query strings.
10. T11: http-transport startup-refusal + auth matrix (reusing issue 38's tests from this suite's mapping) — issue 38 is a dependency, so no skip machinery: all 14 rows are always live.
11. T12: a test asserts the runtime dependency allowlist — exactly `@modelcontextprotocol/sdk`, `zod`, `commander`, `tsshogi` at the top level, with the full transitive closure snapshotted in `test/security/fixtures/dep-tree.json` (regenerated deliberately via a documented npm script; drift fails CI).
12. T13: control-character injection through action fields → stderr capture shows sanitized output.
13. T14: child-process stdio test (13 reuse) asserting stdout purity under error conditions (force an internal error; stack goes to stderr).

## Acceptance Criteria

- [ ] All 14 threats mapped with live tests (no pending/skip states); the meta-test enforces the mapping's completeness.
- [ ] Suite green on CI matrix; total runtime ≤ 90 s.
- [ ] At least the novel tests listed above exist here (not only re-imports): T2 flood, T7-via-32, T8 tripwire, T9 zoo, T10 sentinel, T12 dep allowlist.
- [ ] Deliberately breaking one guard locally (e.g., adding a stray `console.log` in serve path) turns the suite red (verified once during development, noted in PR).

## Validation

- `npm test -- test/security` in CI; PR description includes the T# mapping table.

## Dependencies

- 11, 12, 14, 19, 28, 36, 38.

## Non-goals

- Pen-testing/fuzzing infrastructure, dependency CVE scanning (44), SECURITY.md policy file itself (41).

## Design References

- DESIGN.md §18 (whole), §19.4
