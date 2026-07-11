# mcparcade — v1 Issue Plan

Status: draft for review · Date: 2026-07-11
Derived from [DESIGN.md](./DESIGN.md). Each issue below has a full draft in `docs/issues/NN-short-title.md`; GitHub Issues are generated 1:1 from those drafts and are derived artifacts.

## 1. v1 completion statement

**v1 is complete when all 44 issues below are closed with their Acceptance Criteria and Validation sections satisfied.** At that point the product delivers everything in DESIGN.md §3.1: `npx mcparcade` serves a stdio MCP server (plus optional self-host HTTP) exposing 8 tools; an agent can discover, start, play, resign, and resume maze / 2048 / sudoku / shogi / escape (2 bundled scenarios + custom) / werewolf against built-in deterministic NPCs; results update a persistent local profile with ratings and replays; a token-gated, loopback, read-only spectator web UI shows live games and replay playback; the security model of §18 is implemented and regression-tested; and the package is releasable to npm under MIT with CI, CodeQL, and provenance. Only newly discovered implementation unknowns (§8 below) may add issues.

## 2. Issue list (recommended execution order)

Size: S ≈ half day, M ≈ 1 day, L ≈ 2 days for a focused implementation agent.

| # | File | Title | Size |
|---|---|---|---|
| 01 | `issues/01-project-scaffold.md` | Project scaffold: package, TS/ESM, build, test, lint | M |
| 02 | `issues/02-ci-pipeline.md` | CI pipeline: lint, typecheck, tests on PR | S |
| 03 | `issues/03-core-types-registry.md` | Core domain types, error taxonomy, game registry | M |
| 04 | `issues/04-seeded-rng.md` | Deterministic RNG (splitmix64 + xoshiro128**) with test vectors | S |
| 05 | `issues/05-storage-json-store.md` | Data dir, atomic JSON store, quarantine, advisory lock | M |
| 06 | `issues/06-session-replay-stores.md` | Session & replay stores with schema validation and pruning | M |
| 07 | `issues/07-profile-store-stats.md` | Profile store and per-game stats aggregation | M |
| 08 | `issues/08-rating-elo.md` | Elo rating module with NPC anchors | S |
| 09 | `issues/09-runtime-orchestrator.md` | Game runtime: act loop, NPC resolution, limits, finalization, replayToTurn | L |
| 10 | `issues/10-mcp-catalog-tools.md` | MCP server bootstrap + `list_games` / `describe_game` | M |
| 11 | `issues/11-mcp-session-tools.md` | `start_game` / `observe` / `act` tools | M |
| 12 | `issues/12-mcp-record-tools.md` | `list_sessions` / `get_record` / `get_replay` tools | M |
| 13 | `issues/13-cli-entry.md` | CLI entry: `serve` command, flags, env, stdio wiring | M |
| 14 | `issues/14-integration-test-harness.md` | In-memory MCP integration harness + golden-transcript helper | M |
| 15 | `issues/15-game-maze.md` | Maze game plugin (generator, fog, path action, scoring) | M |
| 16 | `issues/16-game-2048.md` | 2048 game plugin | M |
| 17 | `issues/17-game-sudoku.md` | Sudoku game plugin (bank-backed, check action, scoring) | M |
| 18 | `issues/18-sudoku-puzzle-bank.md` | Sudoku puzzle bank generator tool + committed bank data | M |
| 19 | `issues/19-spectator-http-core.md` | Spectator HTTP core: loopback, token auth, security headers, static assets | L |
| 20 | `issues/20-spectator-live-updates.md` | SSE endpoint + in-proc bus + fs.watch cross-process updates | M |
| 21 | `issues/21-spectator-ui-shell.md` | SPA shell: session list, generic live session view, event log | L |
| 22 | `issues/22-spectator-replay-viewer.md` | Replay list + step/play viewer with frame recompute API | M |
| 23 | `issues/23-spectator-grid-renderers.md` | Pretty grid renderers for maze / 2048 / sudoku | M |
| 24 | `issues/24-shogi-rules-adapter.md` | Shogi rules adapter over tsshogi + edge-rule acceptance tests | L |
| 25 | `issues/25-shogi-npc-engine.md` | Shogi NPC engine: node-capped negamax, eval, difficulties | L |
| 26 | `issues/26-game-shogi-plugin.md` | Shogi game plugin: observe/act/render, outcomes, rating hookup | M |
| 27 | `issues/27-spectator-shogi-renderer.md` | Shogi board renderer in spectator UI | M |
| 28 | `issues/28-escape-scenario-schema.md` | Escape scenario schema, validator library, `validate-scenario` CLI | M |
| 29 | `issues/29-escape-engine.md` | Escape engine: verbs, conditions/effects interpreter, hints, scoring | L |
| 30 | `issues/30-escape-scenario-tutorial.md` | Bundled tutorial scenario `intro-cell` | S |
| 31 | `issues/31-escape-scenario-main.md` | Bundled main scenario `locked-lab` | M |
| 32 | `issues/32-escape-custom-scenarios.md` | Custom scenario loading from data dir | S |
| 33 | `issues/33-werewolf-state-machine.md` | Werewolf phase state machine, roles, win conditions | L |
| 34 | `issues/34-werewolf-speech-protocol.md` | Speech-act schemas and public log model | S |
| 35 | `issues/35-werewolf-npc-engine.md` | NPC suspicion model, personas, wolf/seer strategies | L |
| 36 | `issues/36-game-werewolf-plugin.md` | Werewolf plugin wiring: views, night/day inputs, reveal rules | L |
| 37 | `issues/37-spectator-werewolf-view.md` | Werewolf spectator view: public log, phase timeline, post-game reveal | M |
| 38 | `issues/38-http-transport-selfhost.md` | Streamable HTTP transport: bearer token, binds, rate limit | M |
| 39 | `issues/39-security-test-suite.md` | Cross-cutting security regression suite (§18 threats) | M |
| 40 | `issues/40-determinism-replay-suite.md` | Determinism & replay-invariant suite across all games | M |
| 41 | `issues/41-docs-readme-quickstart.md` | README overhaul + per-client setup guides | M |
| 42 | `issues/42-docs-escape-authoring-guide.md` | Escape scenario authoring guide | S |
| 43 | `issues/43-npm-packaging-release.md` | npm packaging, release workflow with provenance, 0.1.0 checklist | M |
| 44 | `issues/44-repo-security-infra.md` | CodeQL, Dependabot, actions pinning, manual repo-settings checklist | S |

## 3. Dependency table

`A ← B` means B depends on A. Transitive deps omitted.

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01 |
| 04 | 01 |
| 05 | 01, 03 |
| 06 | 03, 05 |
| 07 | 03, 05 |
| 08 | 03 |
| 09 | 03, 04, 06, 07, 08 |
| 10 | 03, 08 |
| 11 | 09, 10 |
| 12 | 06, 07, 09, 10 |
| 13 | 10, 11, 12 |
| 14 | 11, 12 |
| 15 | 04, 09, 14 |
| 16 | 04, 09, 14 |
| 17 | 04, 09, 14, 18 |
| 18 | 01, 04 |
| 19 | 03, 06 |
| 20 | 06, 19 |
| 21 | 19, 20 |
| 22 | 06, 09, 21 |
| 23 | 15, 16, 17, 21 |
| 24 | 01, 03 |
| 25 | 04, 24 |
| 26 | 09, 14, 24, 25 |
| 27 | 21, 26 |
| 28 | 03, 13 |
| 29 | 09, 14, 28 |
| 30 | 28, 29 |
| 31 | 28, 29 |
| 32 | 05, 28, 29 |
| 33 | 03, 04, 34 |
| 34 | 03 |
| 35 | 04, 33, 34 |
| 36 | 09, 14, 33, 34, 35 |
| 37 | 21, 36 |
| 38 | 13 |
| 39 | 11, 12, 14, 19, 28, 36, 38 |
| 40 | 09, 15, 16, 17, 26, 29, 36 |
| 41 | 13, 21, 26, 31, 36, 38 |
| 42 | 28, 29, 30, 31 |
| 43 | 02, 39, 40, 41 |
| 44 | 01, 02, 43 |

## 4. Implementation waves

Issues within a wave are parallelizable unless the table above says otherwise.

| Wave | Issues | Goal / gate to next wave |
|---|---|---|
| **0 — Foundations** | 01–14 | `npx` dev build serves 8 tools end-to-end with zero games (catalog empty), harness green. Gate: integration harness can start/reject/list against a stub game. |
| **1 — Small games** | 15–18 | First real games prove registry→runtime→storage→record pipeline. Gate: golden transcripts for maze/2048/sudoku pass; profile updates verified. |
| **2 — Spectator** | 19–23 | Human can watch wave-1 games live and replay them. Gate: manual smoke + security headers tests green. |
| **3 — Shogi** | 24–27 | Rated shogi vs 3 difficulties, kanji render, board in spectator. Gate: edge-rule acceptance tests (nifu/uchifuzume/sennichite) green; U1/U2 resolved or spawned as issues. |
| **4 — Escape** | 28–32 | Escape engine + 2 scenarios + custom loading + validator CLI. Gate: both scenarios completable via golden transcripts; validator rejects the documented bad-fixture set. |
| **5 — Werewolf** | 33–37 (execute 34 → 33 → 35 → 36 → 37; 33 depends on 34's schemas) | Full werewolf vs NPC lobby with honest hidden info. Gate: leak tests green; scripted game reaches both win conditions. |
| **6 — Ship** | 38–44 | HTTP self-host, security/determinism suites, docs, packaging, repo hardening. Gate: release checklist (§19.5) fully green. |

Waves 2–5 have no cross-dependencies except spectator renderers (23/27/37) and may be reordered or interleaved; the listed order front-loads human-visible value.

## 5. Coverage table (DESIGN.md → issues)

| DESIGN.md section | Covered by issues |
|---|---|
| §4 Architecture / source layout | 01, 03 |
| §5 Tool surface | 10, 11, 12 |
| §6 Domain model | 03 |
| §7 Runtime, limits, errors | 09 (11 for facade mapping) |
| §8 Determinism & RNG | 04, 40 |
| §9 Storage | 05, 06, 07 |
| §10 Records & ratings | 07, 08, 12 |
| §11 Spectator | 19, 20, 21, 22, 23, 27, 37 |
| §12 Small games | 15, 16, 17, 18 |
| §13 Shogi | 24, 25, 26, 27 |
| §14 Escape | 28, 29, 30, 31, 32, 42 |
| §15 Werewolf | 33, 34, 35, 36, 37 |
| §16 CLI | 13 (28 adds `validate-scenario`) |
| §17 HTTP transport | 38 |
| §18 Security model | 39 (plus per-issue security ACs in 05, 10–13, 19, 28, 32, 36, 38) |
| §19 Testing strategy | 14, 39, 40 (plus every issue's Validation section) |
| §20 Packaging & infra | 01, 02, 43, 44 |
| §2 Journeys / onboarding | 41 |
| §21 Known unknowns | tracked in §8 of this plan; each U-item names its resolving issue (U8 resolves via the golden-transcript render-size assertions in issues 15–17, 26, 29, 36) |

Every normative DESIGN.md section maps to at least one issue; no v1 behavior lives only in prose.

## 6. Validation strategy (whole product)

1. **Per-issue gates**: each issue's Validation section is CI-runnable (vitest) except where marked manual; an issue is not done until its named tests exist and pass.
2. **Harness-first rule**: all tool-level behavior is tested through the in-memory MCP harness (issue 14), never by poking internals — this keeps the tool surface honest.
3. **Golden transcripts**: one scripted, seeded, full playthrough per game (win path) + one loss/edge path, snapshot-tested (issues 15–17, 26, 29–31, 36).
4. **Determinism suite** (issue 40): replays every golden game from `(seed, options, actionLog)` and asserts bit-identical states/events; runs on ubuntu + macos in CI.
5. **Security suite** (issue 39): regression tests mapped 1:1 to DESIGN.md §18.2 threats T1–T14.
6. **Manual release checklist** (issue 43): fresh-machine `npx` smoke, Claude Code + Claude Desktop walkthrough, spectator UX pass, `npm pack` file audit.

## 7. Deferred to v2 (not planned as issues)

LLM-NPC adapter (user-supplied key) · agent-vs-agent / human lobbies · hosted multi-tenant service + OAuth · learning loop / strategy-memory files · minesweeper, tsume shogi, handicap shogi, poker · Japanese content & i18n · USI external engines · spectator omniscient mode · MCP resources/prompts surface · replay export/sharing · achievements. (DESIGN.md §3.3.)

## 8. Known unknowns → possible new issues

| # | Unknown (DESIGN.md §21) | Resolution point | If it bites, expected new issue |
|---|---|---|---|
| U1 | tsshogi edge-rule coverage | Issue 24 tests | "Implement missing shogi edge rules in adapter" |
| U2 | Shogi NPC strength/latency balance | Issue 25 calibration | "Retune shogi eval/node caps" |
| U3 | Hard-sudoku generator yield | Issue 18 | "Curate hard sudoku bank manually" |
| U4 | Werewolf NPC believability | Issue 35/36 playtests | "Tune suspicion weights & personas" (constants-only) |
| U5 | `fs.watch` reliability (macOS/Linux edge) | Issue 20 | "Polling fallback for spectator watch" |
| U6 | MCP SDK minor drift at implementation time | Issue 01 install | "Adapt server wiring to SDK x.y" |
| U7 | claude.ai custom-connector auth expectations | Issue 38 manual check | "Adjust HTTP auth/allow-host for claude.ai" |
| U8 | Observation token-cost in long games | Golden transcripts | "Add compact render mode" |

New issues discovered during implementation must be added to this plan and drafted in `docs/issues/` before being opened on GitHub.
