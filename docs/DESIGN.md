# mcparcade — v1 Design

Status: draft for review · Date: 2026-07-11 · License: MIT
Canonical source of truth for requirements and design. Issues in `docs/issues/` reference sections here as `DESIGN.md §N`.

---

## 1. Overview

**mcparcade** is a single MCP (Model Context Protocol) server that turns any MCP-capable AI agent — Claude Code, Claude Desktop, Codex CLI, Cursor, claude.ai custom connectors, or anything else that speaks MCP — into *an AI that can play games*.

The product is explicitly **not** a game with an embedded AI (no Agent SDK, no LLM calls from the server). The user's own agent is the player; mcparcade supplies:

1. **Games** — deterministic rules engines with built-in non-LLM opponents/NPCs.
2. **A uniform, LLM-friendly tool surface** — a handful of generic tools (`list_games`, `start_game`, `observe`, `act`, …) that work identically across all games.
3. **A persistent player identity** — local match records, ratings, and replays, so the agent measurably "grows" as a game player.
4. **A human spectator surface** — a local, read-only web UI where the human watches their agent play live and reviews replays.

v1 games: a **small-games pack** (maze, 2048, sudoku), **shogi**, an **escape game** engine with bundled scenarios, and **werewolf** (social deduction vs. rule-based NPCs).

### 1.1 Design pillars

| Pillar | Consequence |
|---|---|
| The agent is the player | Server never calls an LLM; NPCs are engines/heuristics (ADR-002) |
| Zero-friction install | `npx mcparcade` and one client-config entry; no API keys, no external binaries |
| Deterministic & replayable | All randomness from a per-session seed; session = seed + action log (ADR-004) |
| Honest hidden information | Per-player views computed server-side; full state never leaves the runtime |
| Offline & secretless | No outbound network requests at runtime; nothing sensitive to store |
| Secure by default | Loopback-only spectator with token; validated inputs; capped resources (§18) |

## 2. Users and journeys

### 2.1 Personas

- **P1 — Agent owner (primary)**: runs Claude Code / Codex CLI / Claude Desktop; adds mcparcade to their MCP config; tells their agent "play a game of shogi" or "escape the lab"; watches in the spectator UI.
- **P2 — Chat user**: uses an MCP-capable chat client (Claude Desktop; later claude.ai via self-hosted HTTP). Same flows, no terminal.
- **P3 — Content author (secondary)**: writes a custom escape scenario JSON, validates it with `mcparcade validate-scenario`, drops it in the data dir.
- **P4 — Self-hoster (secondary)**: runs `mcparcade --http 8765` on a personal machine/VPS to connect a remote MCP client. Single-user; bearer token.

### 2.2 Core journey (P1)

1. User adds `{"command": "npx", "args": ["-y", "mcparcade"]}` to their client's MCP config.
2. Agent calls `list_games` → sees catalog. Calls `describe_game("shogi")` → rules, options, action format, examples.
3. Agent calls `start_game("shogi", {difficulty: "normal"})` → gets `sessionId`, first observation, `spectatorUrl`.
4. Agent surfaces `spectatorUrl` to the user; user opens it in a browser and watches live.
5. Loop: agent reads observation (text board + structured state + legal actions) → calls `act` → NPC replies within the same call → new observation.
6. Game ends → outcome recorded, rating updated, replay saved. Agent calls `get_record` to report its stats; user reviews the replay in the spectator UI.

## 3. Scope

### 3.1 v1 scope

- Single npm package `mcparcade` (TypeScript, Node ≥ 20, ESM), bin `mcparcade`.
- MCP server over **stdio** (default) and **Streamable HTTP** (self-host flag, single-user bearer token).
- 8 MCP tools (§5). Tools only — no MCP resources/prompts in v1.
- Game runtime: sessions, per-player views, NPC turn loop, universal actions, limits (§7).
- Determinism: seeded RNG, event-sourced sessions, reproducible replays (§8).
- Persistence in a local data dir: sessions, replays, profile/ratings (§9, §10).
- Spectator web UI: loopback HTTP + SSE, token-gated, read-only; live view + replay playback (§11).
- Games: maze, 2048, sudoku (§12); shogi vs. built-in engine (§13); escape engine + 2 bundled scenarios + custom scenario loading (§14); werewolf vs. rule-based NPCs (§15).
- CLI: `serve` (default), `validate-scenario`; flags/env config (§16).
- Security model implemented as listed in §18; security test suite (§19.4).
- OSS packaging: MIT, CI, CodeQL/Dependabot, npm publish with provenance (§20).

### 3.2 v1 non-goals (explicit)

- No LLM API calls from the server; no LLM-driven NPCs.
- No multi-agent or human-vs-agent play; exactly one human-side player (the connected agent) per session.
- No hosted/multi-tenant service; HTTP mode is single-user self-hosting.
- No external game engines (no USI engine spawning), no accounts, no telemetry, no auto-update, **no outbound network requests at runtime**.
- No i18n framework: agent-facing text is English; shogi renders use standard kanji pieces.
- No achievements/quests; no in-browser play (spectator is read-only).
- No jishogi (impasse) adjudication in shogi; no shogi handicap games; no tsume mode.
- No plugin loading of third-party *code* (custom escape scenarios are declarative data only).

### 3.3 v2 candidates (deferred)

LLM-NPC adapter (user-supplied key), agent-vs-agent lobbies, hosted multi-tenant service with OAuth, learning loop (strategy memory files with efficacy tracking), more games (minesweeper, tsume shogi, handicap shogi, poker), i18n (Japanese content), USI external engines, spectator omniscient mode, MCP resources for rules/replays, replay export/sharing, achievements.

### 3.4 Requirement decisions already made (with the user, 2026-07-10)

| Decision | Choice |
|---|---|
| Delivery | Local stdio first; transport-agnostic core; HTTP self-host in v1; official hosting v2 |
| v1 games | All four categories (small pack, shogi, escape, werewolf) |
| Opponents | Built-in engines / rule-based NPCs; no API keys (ADR-002) |
| Growth | Records/ratings/replays persisted locally; learning loop deferred (v2) |
| Server shape | One server, uniform tool surface, games as internal plugins (ADR-001) |
| Spectator | Local web UI in v1 (read-only) |
| Stack | TypeScript/Node, official MCP SDK |
| License | MIT |

## 4. Architecture overview

```
┌────────────────────────────  process: mcparcade  ───────────────────────────┐
│                                                                              │
│  MCP transports                Core                         Spectator        │
│  ┌──────────────┐    ┌────────────────────┐    ┌──────────────────────────┐  │
│  │ stdio (dflt) │    │ ToolFacade (§5)    │    │ HTTP server (loopback)   │  │
│  │ streamable   ├───▶│  ↓                 │    │  static SPA (bundled)    │  │
│  │ HTTP (flag)  │    │ Runtime (§7)       │◀───│  read-only JSON API      │  │
│  └──────────────┘    │  sessions, turns,  │    │  SSE live events         │  │
│                      │  NPC loop, limits  │    └────────────▲─────────────┘  │
│                      │  ↓          ↓      │                 │ file watch     │
│                      │ GameRegistry Rating│                 │ + in-proc bus  │
│                      └──────┬───────┬─────┘                 │                │
│   Game plugins (in-repo)    │       │        Storage (§9)   │                │
│  ┌──────────────────────────▼─┐   ┌─▼──────────────────────┴─────────────┐  │
│  │ maze │2048│sudoku│shogi│   │   │ ~/.mcparcade/                        │  │
│  │ escape│werewolf  (§12–§15) │   │  profile.json  sessions/  replays/   │  │
│  └────────────────────────────┘   │  scenarios/ (user content)           │  │
│                                    └──────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Source layout (single package)

```
src/
  index.ts                 # CLI entry (commander): serve (default), validate-scenario
  server/
    tools.ts               # MCP tool registration (facade over runtime)
    stdio.ts               # stdio transport wiring
    http.ts                # Streamable HTTP transport wiring (self-host)
  core/
    types.ts               # GameDefinition, Observation, ActResult, events, errors
    registry.ts            # static game registry
    runtime.ts             # session orchestration (§7)
    rng.ts                 # seeded RNG (§8)
    ulid.ts                # in-repo ULID
    errors.ts              # error taxonomy (§7.5)
    limits.ts              # caps & quotas (§7.4)
  storage/
    paths.ts               # data-dir resolution, id→path mapping (traversal-safe)
    jsonstore.ts           # atomic read/write, schema validation, quarantine
    sessions.ts  replays.ts  profile.ts
    lock.ts                # advisory lockfile for profile read-modify-write
  rating/
    elo.ts  stats.ts
  spectator/
    server.ts              # http server, auth, security headers
    api.ts                 # /api routes (read-only)
    sse.ts                 # event stream
    watch.ts               # fs watch → event bus
    assets/                # bundled SPA (no external resources)
  games/
    maze/  g2048/  sudoku/  shogi/  escape/  werewolf/
content/                   # bundled data (JSON) at repo root, shipped in the npm package
  sudoku/  escape-scenarios/
test/
  unit/  integration/  security/  determinism/
tools/
  gen-sudoku-bank.ts       # offline puzzle bank generator (committed data)
```

### 4.2 Component responsibilities

| Component | Owns | Must not |
|---|---|---|
| ToolFacade | MCP schemas, arg validation, mapping errors→tool results | contain game logic |
| Runtime | session lifecycle, turn order, NPC loop, limits, finalization | know transport or HTTP |
| GameDefinition (plugin) | rules, state, views, NPC policy, renders | do I/O, read clocks, call `Math.random` |
| Storage | atomic persistence, schema versions, quarantine | interpret game state |
| Spectator | read-only projection of storage + live events | mutate anything; see hidden info pre-reveal |

## 5. MCP tool surface

Eight tools. All tool inputs are validated with zod; all responses return **both** a human-readable `text` content block (compact) and `structuredContent` (the JSON payloads below). Field names are camelCase. Sizes: any single tool response's text render ≤ 16 KiB; structured payload ≤ 64 KiB.

### 5.1 `list_games`

Input: `{}` → Output:

```json
{ "games": [{
    "gameId": "shogi",
    "title": "Shogi",
    "category": "board",
    "players": "you vs built-in engine",
    "summary": "Japanese chess vs a built-in engine with 3 difficulty levels.",
    "estMinutes": 20,
    "options": ["difficulty", "yourSide", "seed"]
  }, ...],
  "spectatorUrl": "http://127.0.0.1:PORT/?token=... | null" }
```

### 5.2 `describe_game`

Input: `{ gameId }` → Output: `{ gameId, rules, actions, options, examples, tips }` where `rules` is markdown (how to play, win conditions), `actions` documents each action `type` with its JSON shape, `options` is a JSON-Schema-ish description of `start_game` options, `examples` shows 2–3 full `act` payloads, `tips` gives strategy hints for a first-time player. Unknown `gameId` → `E_GAME_NOT_FOUND`.

### 5.3 `start_game`

Input: `{ gameId, options?: object }` — `options` validated against the game's options schema; every game accepts optional `seed` (string, `^[0-9a-f]{1,16}$`).
Output: `{ sessionId, observation, spectatorUrl }`. Limits: max **20 active sessions** total; starting beyond that → `E_TOO_MANY_SESSIONS` (agent should `act {"type":"resign"}` or abandon old ones).

### 5.4 `observe`

Input: `{ sessionId, verbose?: boolean }` → Output: `{ observation }` (§6.4). `verbose: true` includes full legal-action lists where the default is a summarized form. Never mutates state. Unknown/foreign id → `E_SESSION_NOT_FOUND`.

### 5.5 `act`

Input: `{ sessionId, action: object }` (≤ 8 KiB JSON).
Output:

```json
{ "accepted": true,
  "events": [ {"t": 41, "kind": "npc_move", "playerId": "p2", "summary": "△3四歩", "data": {...}}, ... ],
  "observation": { ... },
  "outcome": null | { ... §6.6 } }
```

Rejected action → `accepted: false`, plus `error: {code, message, hint}` and an unchanged observation; **state is never modified by a rejected action**. Universal actions handled by the runtime for every game: `{"type":"resign"}` (finalize as loss/abandon per game rules). Games may define `{"type":"hint"}` (escape only in v1).

### 5.6 `list_sessions`

Input: `{ status?: "active"|"finished"|"all", gameId? }` → Output: `{ sessions: [{sessionId, gameId, status, turn, updatedAt, summary}], spectatorUrl }`. Default `status: "active"`. Max 50 rows, newest first.

### 5.7 `get_record`

Input: `{ gameId? }` → Output: profile summary (§10.1): per-game stats, ratings, streaks, `recentReplays: [{replayId, gameId, outcome, endedAt}]` (≤ 10 per game).

### 5.8 `get_replay`

Input: `{ replayId, from?: number, to?: number }` → Output: replay metadata + a paginated slice of **move entries** `[{t, playerId, action, summary}]` (the action-log entry joined with its same-turn event summary; ≤ 500 entries per call, paginate with `from`/`to`, response carries `total`). Includes the final outcome.

### 5.9 Tool-surface rules

- Tool names are stable API; renaming is a breaking change (semver).
- Every error returned to the agent carries `code` from §7.5, a one-line `message`, and an actionable `hint` (e.g. "call observe and pick one of legalActions").
- `spectatorUrl` is `null` when the spectator is disabled. The token is embedded in the URL; the agent should show the URL to the human verbatim.

## 6. Core domain model

### 6.1 `GameDefinition` (the plugin contract)

```ts
interface GameDefinition<S, A> {
  id: GameId;                    // "maze" | "g2048" | "sudoku" | "shogi" | "escape" | "werewolf"
  title: string;
  version: string;               // bump on any state/action schema change; stored in sessions
  category: "puzzle" | "board" | "social";
  summary: string; estMinutes: number;
  rulesDoc: string;              // markdown for describe_game
  optionsSchema: z.ZodType;      // start options (incl. optional seed)
  actionSchema: z.ZodType;       // discriminated union on action.type
  maxTurns: number;              // hard cap (§7.4)
  scoring: "winloss" | "score";  // drives rating vs best-score records (§10)

  init(opts: ValidatedOptions, rng: Rng): InitResult<S>;   // players incl. NPCs
  observe(state: S, viewer: PlayerId, verbose: boolean): GameView;
  act(state: S, player: PlayerId, action: A, rng: Rng): ActOutcome<S>;  // pure: returns new state + events, or rejection
  currentPlayer(state: S): PlayerId | null;                // whose input the runtime collects next; null ONLY when finished. Simultaneous phases return the agent id while its input is pending (the plugin resolves NPC inputs internally, §15.5)
  terminal(state: S): Outcome | null;
  npcAct?(state: S, npc: PlayerId, rng: Rng): A;           // required if init creates NPC players
  renderWeb?(state: S, viewer: "public" | PlayerId): WebRenderModel;  // §11.6
}
```

Purity rules (enforced by review + determinism tests): plugins never touch `Date`, `Math.random`, filesystem, or network; all randomness flows through the injected `Rng`; identical `(state, action, rng-stream)` must yield identical results on any machine.

### 6.2 Players

`players: [{ id: "p1", kind: "agent" } | { id: "p2", kind: "npc", npcProfile: string }]`. Exactly one `agent` player in v1. Ids are `p1..pN`, assigned by `init`.

### 6.3 Events

Everything that happens is an event appended to the session's public/typed log:

```ts
{ t: number,                // 1-based global turn counter
  kind: string,             // e.g. "move", "npc_move", "phase", "speech", "death", "info"
  playerId?: PlayerId,
  visibility: "public" | PlayerId,   // private events delivered only to that player
  summary: string,          // one line, human/agent readable
  data?: object }           // typed per game
```

`act` responses include all events with `visibility` public-or-you since the agent's previous action/observe.

### 6.4 Observation (returned by `observe` / inside `act`)

```json
{ "sessionId": "sess_01J...", "gameId": "shogi", "gameVersion": "1.0.0",
  "status": "your_turn" | "finished",
  "turn": 41, "maxTurns": 400,
  "you": { "playerId": "p1", "role": "seer|null", "private": { ... } },
  "render": "text board / scene, monospace-safe, ≤ 16 KiB",
  "state": { ...game-specific structured public+your-private view... },
  "legalActions": { "mode": "list", "actions": [...] }
                  | { "mode": "schema", "summary": "place(row 1-9, col 1-9, digit 1-9)", "count": 231 },
  "events": [ ...since your last observation... ],
  "outcome": null | { ... } }
```

Rules: the runtime never returns an observation while an NPC still has to move (`act` resolves NPC turns before returning, §7.2); `status` is only ever `your_turn` or `finished` from the agent's perspective. `legalActions.mode` is `"list"` when the count ≤ 64 or `verbose: true` (cap 512), else `"schema"`.

### 6.5 ActOutcome

`{ ok: true, state, events } | { ok: false, code, message, hint }` — rejections must not mutate state.

### 6.6 Outcome

```json
{ "result": "win" | "loss" | "draw" | "abandoned" | "aborted_turn_limit",
  "reason": "checkmate | resign | village_win | escaped | ...",
  "score": 8420 | null,        // score games
  "par": {...} | null,          // escape
  "ratingDelta": +12 | null,    // winloss games
  "rated": true,                // optional, default true; false for practice/abandoned/turn-limit (§10.2)
  "summaryExtras": {"optimality": 92} | null,   // game-specific profile extras (§10.3)
  "summary": "one-line result" }
```

## 7. Runtime orchestration

### 7.1 Session lifecycle

```
start_game ─▶ active ──act/observe──▶ active
                │ resign / terminal / turn-cap / log-cap
                ▼
             finalizing ─▶ finished (outcome, rating update, replay extract)
```

`abandoned`: `resign` before turn 5 in winloss games records `abandoned` (no rating change); later resigns are `loss` (shogi) or per-game rule. Sessions `active` and untouched for > 30 days are treated as `abandoned` lazily on next load (no background jobs).

### 7.2 The `act` algorithm (normative)

```
1  load session (E_SESSION_NOT_FOUND if missing; E_SESSION_FINISHED if not active)
2  assert currentPlayer(state) == "p1" (else E_NOT_YOUR_TURN; §6.1: simultaneous phases report "p1" while the agent's input is pending)
3  parse action: runtime universal actions first (resign), else game.actionSchema (E_INVALID_ACTION on schema fail)
4  outcome = game.act(state, "p1", action, rng)      // rejection → return accepted:false, no writes
5  append event(s); state = outcome.state
6  while terminal(state) == null and currentPlayer(state) is an NPC:
       a = game.npcAct(state, npc, rng)
       apply game.act(state, npc, a, rng)            // npc rejection = plugin bug → E_INTERNAL, session frozen
       append events   (loop guard: > 200 NPC steps per act call → E_INTERNAL)
7  if terminal(state) != null → finalize (§7.6)
8  persist session atomically (§9.3); on finalize also update profile + write replay
9  return { accepted, events(visible to p1), observation(p1), outcome? }
```

Werewolf's simultaneous phases (night actions, votes) are modeled inside the plugin: the agent submits its input, the plugin immediately resolves NPC inputs in the same step (§15.5), so the runtime loop above still holds.

`start_game` runs the same NPC-resolution loop (step 6) immediately after `init`, so when an NPC moves first (e.g. shogi with the agent as white) the first observation already reflects the NPC's opening move.

### 7.3 Invalid-action ergonomics

Rejections are cheap and expected (LLM players). Response includes `hint` and, for list-mode games, up to 10 sample legal actions. The runtime counts consecutive rejections per session; after 10 in a row it appends the full legal-action list (or schema) to the next error. Rejections do not consume turns.

### 7.4 Limits (defaults; single place `core/limits.ts`)

| Limit | Value | On breach |
|---|---|---|
| Active sessions | 20 | `E_TOO_MANY_SESSIONS` |
| `action` JSON size | 8 KiB | `E_INVALID_ACTION` |
| Free-text field (`flavor`, etc.) | 1 KiB | `E_INVALID_ACTION` |
| Turns per session | per-game `maxTurns` (§12–§15) | finalize `aborted_turn_limit` |
| Events per session | 20 000 | finalize `aborted_turn_limit` |
| Session file size | 2 MiB | finalize `aborted_turn_limit` |
| NPC steps per `act` | 200 | `E_INTERNAL` (freeze session) |
| Replay page | 500 entries | paginate |

### 7.5 Error taxonomy

`E_GAME_NOT_FOUND · E_SESSION_NOT_FOUND · E_REPLAY_NOT_FOUND · E_SESSION_FINISHED · E_NOT_YOUR_TURN · E_INVALID_ACTION · E_INVALID_OPTIONS · E_TOO_MANY_SESSIONS · E_STORAGE (I/O failure; retry-safe) · E_CORRUPT_SESSION (quarantined, §9.4) · E_INTERNAL (bug; session frozen read-only)`. Errors are returned as successful MCP tool calls with `isError: false` and a structured `error` object — the agent must be able to read and react to them; transport-level errors are reserved for malformed tool input that fails zod validation (SDK behavior).

### 7.6 Finalization & logging

On terminal: compute `Outcome`, update stats/rating (§10), extract replay (§9.5), mark `finished`, emit `session-finished` bus event (spectator). All logging (including finalization) goes to **stderr** with level prefixes (`info|warn|error`); control characters in any user/agent-supplied string are stripped before logging (log-injection hygiene, §18).

## 8. Determinism & RNG

- Master seed: 64-bit, hex string. Default: generated from `crypto.randomBytes(8)` at `start_game`; overridable via `options.seed` (recorded either way).
- Streams: `splitmix64(masterSeed ⊕ streamTag)` seeds one **xoshiro128\*\*** instance per purpose: `init`, `npc:p2`, `npc:p3`, …, `game`. Stream tags are fixed FNV-1a-64 hashes of the literal strings. The exact constants, state mapping, uint32 arithmetic, `[0,1)` conversion, and rejection-sampling rules are normative in issue 04 (`docs/issues/04-seeded-rng.md`) and implemented in `core/rng.ts` with published test vectors (10 known values per stream from seed `deadbeefcafef00d`), so replays survive refactors and platforms.
- `Rng` API: `next(): number [0,1)`, `int(loIncl, hiExcl)`, `pick<T>(arr)`, `shuffle<T>(arr)` (Fisher–Yates). Plugins receive only the streams they own.
- Replay invariant (tested in CI, §19.5): `replay(seed, options, actionLog) ≡ final state & events`, on every platform, for every bundled game.
- Non-deterministic things (session ids, timestamps) live outside game state and are excluded from the invariant.

## 9. Persistence & storage

### 9.1 Data directory

Resolution order: `--data-dir` flag → `MCPARCADE_DATA_DIR` env → `~/.mcparcade`. Created on demand with mode `0700` (files `0600`).

```
~/.mcparcade/
  profile.json
  sessions/sess_<ULID>.json
  replays/rep_<ULID>.json
  scenarios/*.json            # user-authored escape scenarios (§14.8)
```

### 9.2 Schemas & versioning

Every file: `{ "schemaVersion": 1, ... }`, validated with zod on read. Unknown newer version → refuse to touch the file (`E_STORAGE`, message tells user to upgrade). Migration policy: additive fields preferred; breaking changes ship a migration in `storage/` and bump `schemaVersion`.

Session file:

```json
{ "schemaVersion": 1, "sessionId": "sess_01J8...", "gameId": "shogi", "gameVersion": "1.0.0",
  "seed": "9f3c2a17b04d55e1", "options": {"difficulty": "normal", "yourSide": "black"},
  "createdAt": "2026-07-11T02:10:00Z", "updatedAt": "...",
  "status": "active|finished|abandoned",
  "players": [{"id":"p1","kind":"agent"},{"id":"p2","kind":"npc","npcProfile":"normal"}],
  "actionLog": [ {"t":1,"playerId":"p1","action":{...},"at":"..."} , ... ],
  "eventLog":  [ ...§6.3 events... ],
  "stateSnapshot": { ...latest full game state (fast resume)... },
  "rngCursors": { "game": 42, "npc:p2": 17 },   // draws consumed per stream; snapshot+cursors = resume point (§7.2 step 8)
  "frozen": false,                               // set on E_INTERNAL; session becomes read-only (§7.5)
  "outcome": null }
```

IDs: `sess_`/`rep_` + Crockford-base32 ULID; validated everywhere with `^(sess|rep)_[0-9A-HJKMNP-TV-Z]{26}$` **before** any path construction (path-traversal guard; §18.4).

### 9.3 Atomicity & concurrency

- Writes: serialize → write `<file>.tmp.<pid>` → `fsync` → `rename` (atomic on POSIX). Never partial files.
- Sessions: one agent drives one session, but to make cross-process races safe every session mutation (`act`, finalization, stale-abandon) is wrapped in a short per-session advisory lock (`withLock(sessionId)`, same mechanism as below) — concurrent `act` calls from two processes serialize instead of losing writes. Spectator only reads.
- Advisory lock mechanism: `<name>.lock` containing pid + timestamp, O_EXCL create, 5 s stale-takeover, 3 retries with 100 ms backoff. Locks are held only around the read-modify-write itself (milliseconds — replay extraction and other I/O happen outside the lock), so the 5 s takeover threshold is ~1000× the expected hold time.
- `profile.json`: read-modify-write under `withLock("profile")`.
- Multiple mcparcade processes (e.g. Claude Code + Claude Desktop) are supported: they share the data dir; each has its own spectator port; spectators watch the filesystem so they see all processes' sessions (§11.5).

### 9.4 Corruption handling

Before reading any data-dir file, `stat` its size: files over 8 MiB are treated as corrupt without being read (hostile-file guard, T9). Unreadable/invalid/oversized JSON → rename to `<name>.corrupt-<ts>`, log to stderr, continue (`E_CORRUPT_SESSION` if it was directly requested). Never crash the server on bad data files; never delete user data.

### 9.5 Replays

On finalization: write `rep_<ulid>.json` = session metadata + `actionLog` + `eventLog` + `outcome` + final `stateSnapshot` (for renderers), minus nothing else. Pruning beyond caps — 200 finished/abandoned session files (oldest by `updatedAt` deleted first), 500 replays (oldest by `endedAt`) — runs after each finalization; active sessions are never pruned (profile keeps aggregate stats forever).

## 10. Records, ratings, replays

### 10.1 `profile.json`

```json
{ "schemaVersion": 1, "createdAt": "...",
  "games": {
    "shogi":  { "plays": 12, "wins": 5, "losses": 6, "draws": 0, "abandoned": 1,
                "rating": 1084, "peakRating": 1121, "streak": -2,
                "byDifficulty": {"easy": {"w":3,"l":0}, "normal": {"w":2,"l":6}},
                "recentReplays": ["rep_...", ...] },
    "g2048":  { "plays": 30, "bestScore": 20120, "avgScore": 6210.5,
                "lastExtras": {"highestTile": 2048},
                "recentReplays": [...] },
    ...
  } }
```

### 10.2 Rating (winloss games: shogi, werewolf)

Elo vs. fixed NPC anchors. Player starts at 1000, floor 400, K = 32 (K = 16 after 30 rated games per game).
`expected = 1/(1+10^((anchor−rating)/400))`; `rating = max(400, Math.round(rating + K * (score − expected)))`, score ∈ {1, 0.5, 0} (JS `Math.round` semantics are normative — see issue 08).
Anchors — shogi: easy 800 / normal 1200 / hard 1600. Werewolf (one rating vs. the NPC table): 1200; win = beat the lobby. Draws (sennichite) score 0.5. `abandoned` games are unrated. Anchor values are product constants (documented in `describe_game`), not claims about human Elo.

### 10.3 Score games (maze, 2048, sudoku, escape)

Track `bestScore`, `avgScore`, `plays`, plus `lastExtras` (the latest outcome's `summaryExtras`, §6.6): maze = `{size, optimality}`, g2048 = `{highestTile}`, sudoku = `{errorsCommitted, checksUsed}`, escape = `{ending, turns, parTurns, hintsUsed}`.

## 11. Spectator web UI (v1, read-only)

### 11.1 Posture

Separate loopback HTTP server inside the same process. **Read-only**: it renders state; it never mutates and exposes no action endpoints. Enabled by default in `serve`; `--no-spectator` disables; `--spectator-port` (default 0 = ephemeral).

### 11.2 AuthN/AuthZ

Per-process random token (`crypto.randomBytes(16)` → 32-hex), never persisted. Every request must carry it: `?token=` for the page bootstrap and for the SSE stream (EventSource cannot set headers), `Authorization: Bearer` / `X-Mcparcade-Token` for JSON API calls (the SPA uses the header after bootstrap). Constant-time compare. 401 otherwise. The page immediately rewrites `location` via `history.replaceState` to drop the token from the address bar/history.

### 11.3 Network hardening (DNS-rebinding etc.)

Bind `127.0.0.1` only. Reject unless `Host` ∈ {`127.0.0.1[:port]`, `localhost[:port]`, `[::1][:port]`}. Reject any request with an `Origin` header not in that set (SSE/fetch from foreign origins). Headers on every response: `Content-Security-Policy: default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store` (API). No CORS headers at all (same-origin only). All assets bundled; zero external requests. Access/error logging redaction: request log lines record the path **without the query string** and never record `Authorization`/token headers (T10).

### 11.4 HTTP API (all GET, all token-gated)

```
GET /                       SPA shell (token-gated via ?token=)
GET /assets/*               bundled static (immutable; Host/Origin-checked but NOT token-gated —
                            <script>/<link> requests cannot carry the token after the URL scrub;
                            assets are public build artifacts containing no user data)
GET /api/meta               {version, games[], watchMode: "watch"|"poll"}
GET /api/sessions?status=   [{sessionId, gameId, status, turn, updatedAt, summary}]
GET /api/sessions/:id       {meta, publicView}        # §11.6; PUBLIC view only while active
GET /api/replays            [{replayId, gameId, outcome, endedAt}]
GET /api/replays/:id        {meta, outcome, frames?}  # full log for playback
GET /api/replays/:id/frame/:t   {renderModel}         # state at turn t (recomputed via §8 replay)
GET /api/events             SSE stream (§11.5)
```

`:id` validated against the ULID regex before path use. JSON only; unknown id → 404 JSON.

### 11.5 Live updates

In-process event bus emits `session-updated`/`session-finished`/`replay-added`; additionally `fs.watch` on `sessions/` + `replays/` (300 ms debounce) so sessions driven by *other* mcparcade processes appear too. SSE framing: `event: <type>` with `data: {"type":"session-updated","sessionId":"...","gameId":"...","turn":n}` (type appears in both places; data is id-and-counter only — the client refetches, state never travels over SSE). Heartbeat comment every 25 s; auto-reconnect via `EventSource` defaults.

### 11.6 Render model (what the SPA draws)

`GET /api/sessions/:id` returns `publicView`:

```json
{ "render": "text render (public viewer)",
  "web": null | { "kind": "grid", "cells": [...], ... } | { "kind": "shogi", ... } | { "kind": "chatlog", ... },
  "panels": { "players": [...], "score": ..., "log": [last 100 public events] } }
```

`web` comes from `GameDefinition.renderWeb(state, "public")`. The SPA always has the `<pre>` text fallback, so per-game pretty renderers are additive (grid for maze/2048/sudoku; board for shogi; chat/phase timeline for werewolf). **Hidden-information rule**: while a session with hidden info (werewolf) is `active`, the public view exposes only public events/state; full reveal appears only after `finished` (§15.8). Escaping: every string the SPA injects goes through one `esc()` helper; renderers build DOM via `textContent`, never `innerHTML` with untrusted strings.

### 11.7 Replay playback

Replay page: step/back/play (1×/4×) over turns; frame states come from `/frame/:t` (server-side deterministic recompute with an LRU cache of 32 frames). Frame recompute requires the replay's `gameVersion` to exactly match the installed game's version; on mismatch the UI shows the stored final `stateSnapshot` and the event log instead of per-turn frames (ADR-004). Werewolf replays: per-turn frames use the public projection (what a spectator saw at that moment); the full reveal (roles, night choices, trajectories) is a separate panel sourced from the replay's final state, unlocked at the final frame or via a spoiler toggle (§15.8, issue 37).

## 12. Small-games pack (maze, 2048, sudoku)

Shared goals: tiny engines, validate the whole pipeline (registry→runtime→storage→spectator) before the big games. All are `scoring: "score"`, single player, no NPC players.

### 12.1 Maze

- Options: `{ size?: 9|15|21 (default 15), fog?: boolean (default false), seed? }`. Odd sizes; generator: recursive backtracker on the `init` rng; entrance top-left `S`, exit bottom-right `E`.
- State: grid, player pos, steps, visited set (fog mode reveals seen cells only).
- Actions: `{"type":"move","dir":"up|down|left|right"}` · `{"type":"path","dirs":[...]}` (≤ 64 steps; stops at first illegal step with rejection) · resign.
- Render: `#` wall, `·` floor, `@` you, `S`/`E`; fog: unseen = ` `.
- Terminal: reach `E` → win. Score = `round(1000 * optimal / steps)` (optimal = BFS shortest path length, computed at init). `maxTurns: 2000`.
- Records: bestScore + `summaryExtras: {size, optimality}` (§10.3).

### 12.2 2048 (`gameId: "g2048"`)

- Options: `{ seed? }`. 4×4, two starting tiles.
- Actions: `{"type":"move","dir":"up|down|left|right"}` — a move that changes nothing is rejected (`E_INVALID_ACTION`, hint lists legal dirs).
- Spawn: standard — random empty cell, 90 % `2` / 10 % `4`, from the `game` rng stream.
- Render: 4×4 box-drawing grid, right-aligned numbers; state includes `grid` (row-major ints), `score`, `moves`, `highestTile`.
- Terminal: no legal move → done (`result: "win"` if 2048 reached else `"loss"`; both keep score). Reaching 2048 emits an event but play continues (standard). `maxTurns: 4000`.

### 12.3 Sudoku

- Options: `{ difficulty?: "easy"|"normal"|"hard" (default normal), puzzleId?, seed? }`. Puzzle chosen from the bundled bank (§12.4) by seed unless `puzzleId` given.
- State: `given` mask, current grid, `errorsCommitted` (count of placements that contradict the unique solution — checked silently, revealed only in final score), turn count.
- Actions: `{"type":"place","row":1-9,"col":1-9,"digit":1-9}` · `{"type":"erase","row","col"}` (givens are immutable → rejection) · `{"type":"check"}` (reports whether current grid has any *rule* conflicts — row/col/box duplicates — without revealing solution digits; costs 1 turn) · resign.
- Terminal: grid full ∧ rule-valid. If it matches the unique solution → win; a full-but-wrong grid is impossible when rule-valid + unique solution. Score = `max(0, 1000 − 10*errorsCommitted − max(0, turns − filledCellsNeeded))`; `summaryExtras: {errorsCommitted, checksUsed}`. `maxTurns: 500`.
- Render: 9×9 with box separators; givens plain, your entries same glyph (state marks which are given).

### 12.4 Sudoku puzzle bank

Committed data `content/sudoku/bank-v1.json`: ≥ 60 puzzles each for easy/normal and ≥ 40 for hard (hard-generation yield is U3), each `{ id, difficulty, givens: "81-char string", solution: "81-char", clues: n }` with **verified unique solutions**, plus a top-level `generator: {seed, generatedAt, tool, version}` provenance object. Produced offline by `tools/gen-sudoku-bank.ts` (backtracking generator + uniqueness check via solution counting; difficulty by clue-count bands — easy ≥ 38 clues, normal 28–32, hard 24–26; technique-based grading deferred to v2). The generator is repo tooling (run by maintainers, committed output), not shipped in the npm package.

## 13. Shogi

### 13.1 Foundation

`tsshogi` (MIT) behind an in-repo `ShogiRules` adapter (see docs/research/shogi-libraries.md). The adapter is the only file importing tsshogi. It exposes: `initialPosition()`, `toSfen(state)`, `legalMoves(state): UsiMove[]`, `apply(state, usi)`, `inCheck`, `isCheckmate`, `repetitionStatus(history)`, plus renders.

### 13.2 Options

`{ difficulty?: "easy"|"normal"|"hard" (default "normal"), yourSide?: "black"|"white"|"random" (default "black"), seed? }`. Black (先手) moves first.

### 13.3 Actions

`{"type":"move","usi":"7g7f"}` — USI format incl. promotion `+` (`8h2b+`) and drops (`P*5e`). Rejection hints: when the USI is well-formed but illegal, the hint explains why if cheaply known (nifu, into-check, no such piece in hand) and shows up to 10 legal moves. `verbose` observation lists all legal moves (typically < 200).

### 13.4 Rules enforcement (acceptance-tested with fixed SFEN positions)

Legal movement & drops; promotion rules (optional/forced); **nifu**; last-rank drop restrictions (pawn/lance/knight); **uchifuzume** (pawn-drop mate is illegal); **sennichite**: 4-fold repetition → draw, perpetual-check repetition → checker loses. Jishogi/impasse: out of scope (§3.2) — turn cap ends the game `aborted_turn_limit`, unrated. `maxTurns: 400` (plies).

### 13.5 Built-in NPC engine (in-repo, deterministic)

Negamax + alpha-beta on the adapter's move list. **Node-capped iterative deepening** (not wall-clock — determinism): easy = depth 2 cap 5 000 nodes, normal = depth 4 cap 50 000, hard = depth 6 cap 400 000; stable move ordering (captures by MVV-LVA, then USI lexicographic); ties broken by the `npc:p2` rng among moves within 30 centipawns of best (easy) / exact-equal only (normal, hard).
Eval: material (P90 L315 N405 S540 G540 B855 R990; promoted ≈ gold except +B 1445/+R 1550 — in-hand pieces at face value), piece-square tables (committed constants), king-safety: −40/attacker within 2 files·ranks of own king, tempo +20. Centipawn ints only (no floats — cross-platform determinism).
Performance target: normal ≤ 1 s, hard ≤ 5 s per move on a 2020 laptop (node caps sized to that; CI asserts node caps, not wall time).

### 13.6 State & render

State (agent view): SFEN, `yourSide`, hands (piece counts both sides), `inCheck`, move history tail (last 10, USI + kanji), `legalActions` per §6.4. Render: 9×9 full-width board with kanji pieces (`歩香桂銀金角飛玉と杏圭全馬龍`), white pieces marked `v` prefix, hands lines (`先手持駒: 歩三 桂`), last move marker `*`.

### 13.7 Outcomes & rating

Checkmate / resign → win/loss; sennichite → draw (0.5); perpetual check → loss for checker; turn cap → `aborted_turn_limit` (unrated). Rated vs. difficulty anchors (§10.2).

## 14. Escape game

### 14.1 Shape

A declarative scenario interpreter (`gameId: "escape"`, `scoring: "score"`). Scenarios are pure data (JSON) — rooms, entities, interactions, conditions, effects. **No code, no eval, no string interpolation beyond static text.**

### 14.2 Scenario schema (v1) — normative

```jsonc
{ "schemaVersion": 1,
  "id": "locked-lab",              // ^[a-z0-9-]{1,32}$
  "title": "The Locked Lab", "author": "…", "lang": "en",
  "brief": "intro text shown at start",
  "par": { "turns": 40, "hints": 0 },
  "start": { "room": "lab", "inventory": [] },
  "flags": ["power_on", "safe_open"],                 // declared up front
  // "Items" are not a separate collection: an item is any entity with portable: true.
  // has_item / add_item / remove_item / use.item reference portable entities' ids
  // (the validator enforces portability on item references).
  "rooms": [ { "id": "lab", "name": "Laboratory",
               "description": "…(fallback look text)…",
               "exits": [ { "dir": "north", "to": "hall", "lockedBy": "door_locked"?, "lockedText": "…" } ] } ],
  "entities": [ { "id": "desk", "room": "lab" | null,   // null = starts nowhere (spawned later)
                  "name": "steel desk", "portable": false,
                  "description": "…(examine text)…",
                  "states": { "current": "closed", "all": ["closed","open"] }?,
                  "interactions": [
                    { "verb": "open", "with": null,
                      "requires": [ {"kind":"flag","flag":"power_on","is":true},
                                    {"kind":"has_item","item":"key"} ],
                      "failText": "It won't budge.",
                      "effects": [ {"kind":"set_entity_state","entity":"desk","state":"open"},
                                   {"kind":"reveal_item","item":"memo","room":"lab"},
                                   {"kind":"set_flag","flag":"safe_open","value":true},
                                   {"kind":"show_text","text":"The drawer slides open."} ],
                      "once": true } ] } ],
  "hints": [ { "tier": 1, "afterTurns": 10, "text": "…", "penalty": 50 } ],   // ordered tiers
  "endings": [ { "id": "escaped", "kind": "win", "text": "…" } ] }
```

Conditions (`requires`, all must hold): `flag`, `has_item`, `entity_state`, `in_room`, `turns_at_least`. Effects (applied in order): `set_flag`, `add_item`, `remove_item`, `reveal_item`, `set_entity_state`, `unlock_exit`, `lock_exit`, `move_player`, `show_text`, `end_game(ending)` — `end_game` terminates effect processing (later effects in the list are not applied). Anything else fails validation. Exact per-kind payload fields are normative in issue 28.

### 14.3 Validation & caps (validator = library + CLI `validate-scenario`)

zod schema plus referential checks: every referenced room/entity/flag/item/ending exists; `start.room` exists; at least one `end_game` win-ending reachable by static graph walk (conditions treated as satisfiable); no duplicate ids. Caps: file ≤ 256 KiB, rooms ≤ 100, entities ≤ 500, interactions/entity ≤ 20, effects/interaction ≤ 20, text field ≤ 2 KiB, hints ≤ 10. Validator output: list of `{level: error|warning, path, message}`.

### 14.4 Actions

`{"type":"look"}` (room) · `{"type":"examine","target":"desk"}` · `{"type":"take","target":"memo"}` · `{"type":"use","item":"key","target":"desk"?}` · `{"type":"open","target":"desk"}` / `{"type":"move","dir":"north"}` · `{"type":"inventory"}` · `{"type":"hint"}` (next tier if `afterTurns` reached; applies penalty) · resign. Targets matched by entity `id` or case-insensitive `name`; ambiguous name → rejection listing candidates. Verbs `open`/`use` resolve to the entity's matching `interaction` (verb + `with` item), checking `requires`; no matching interaction → `failText` or generic "Nothing happens." **as a rejected action (no state change, no turn consumed)**; matched-but-`requires`-failed → `failText` shown, turn consumed (the attempt happened).

### 14.5 Observation & render

Render = current room name, description, visible entities, exits (with locked markers), inventory, turn/par, last effect text. State mirrors that structurally. `legalActions.mode: "schema"` with verb summary + visible target ids.

### 14.6 Scoring & terminal

`score = max(0, 1000 − 10·max(0, turns − par.turns) − Σ hintPenalties)`; the reached ending is recorded via `summaryExtras.ending` (§10.3). Lose-endings and resign → `result: "loss"`, score 0. `maxTurns: 300`.

### 14.7 Bundled scenarios (content issues)

1. `intro-cell` (tutorial, ~6 entities, 2 rooms, par 15): teaches the full verb set — examine/take/use/open/move/inventory/hint.
2. `locked-lab` (main, ~9 rooms, ~30 entities, 2 endings, par 60): multi-step chains (power → code → safe → keycard), one red herring, hint tiers 3.
Authoring guide: docs issue 42.

### 14.8 Custom scenarios

Loaded at startup from `<dataDir>/scenarios/*.json` (at most 50 files, bytewise-ascending basename order; beyond that a warning names the skipped files), each run through the full validator; invalid → skipped with a stderr warning (never crash). `list_games` shows escape scenarios as options (`options.scenario: "locked-lab" | "intro-cell" | "custom:<id>"`). Trust note (§18.7): scenario text is rendered into the agent's context — installing a scenario = trusting its author; the validator caps sizes but does not referee prose.

## 15. Werewolf (人狼)

### 15.1 Setup

`gameId: "werewolf"`, `scoring: "winloss"`. Options: `{ playerCount?: 5|7|9 (default 7), agentRole?: "random"|"villager"|"werewolf"|"seer" (default random; non-random marked "practice", unrated), seed? }`. Role sets: 5p = 1W/1S/3V · 7p = 2W/1S/4V · 9p = 2W/1S/6V. NPCs get distinct names (`Ash, Blake, Casey, Drew, Emery, Flynn, Gale, Harper`) and persona profiles (§15.6). Seat order = rng shuffle at init.

### 15.2 Phase state machine (normative)

```
setup → night(1) → day(2).announce → day(2).discussion(round 1..2)
      → day(2).vote [→ runoff] → day(2).execution → wincheck
      → night(2) → … (day numbering: night(k) precedes day(k+1))
wincheck: wolves_alive == 0 → village win · wolves_alive >= others_alive → wolf win · else next night
```

Night(1) has no kill (first-night protection, standard-lite); seer still divines night 1. `maxTurns: 200` global turns (agent + NPC accepted actions, the standard §7.2 definition); a full 9p game stays ≪ 150 global turns (simulation-asserted in issue 36).

### 15.3 Night phase

Simultaneous-input step. If the agent has a night action it submits: seer `{"type":"divine","target":"Casey"}`; wolf `{"type":"kill","target":"Drew"}` (with 2 wolves, the *lead* wolf — lowest seat alive — chooses; an agent non-lead wolf submits `{"type":"night_pass"}` but sees the chosen target); villager `{"type":"night_pass"}`. Plugin then resolves NPC night choices, applies deaths, advances to day. Seer result: `"werewolf" | "not_werewolf"`, delivered as a private event.

### 15.4 Day phase — discussion

2 rounds; within a round, alive players speak in seat order; when it's the agent's seat, `status: "your_turn"` with `expected: "utter"`. Utterance:

```json
{"type":"utter",
 "acts":[ {"kind":"claim_role","role":"seer"},
          {"kind":"report_divination","target":"Casey","result":"werewolf"},
          {"kind":"accuse","target":"Casey","reason":"contradiction"},
          {"kind":"defend","target":"Drew"},
          {"kind":"declare_vote","target":"Casey"},
          {"kind":"pass"} ],
 "flavor":"free text ≤ 1 KiB, shown in logs, IGNORED by NPC logic"}
```

≤ 4 acts per utterance; `reason` ∈ fixed enum (`contradiction, vote_pattern, claim_timing, defended_wolf, gut`). Structured acts are the *only* channel into NPC reasoning (ADR-005) — the agent cannot prompt-inject NPCs, and NPCs never parse prose.

### 15.5 Day phase — vote

Simultaneous: agent submits `{"type":"vote","target":"Casey"}` (self-vote/dead target rejected); plugin computes NPC votes, reveals all at once. Plurality is executed; tie → single runoff among tied (revote); tie again → rng among tied (`game` stream). Execution reveal: dead player's role becomes public (standard reveal rules).

### 15.5b Agent death

If the agent dies (night kill or execution), the game plays out to its natural end within that same `act` call via the NPC loop; the agent receives the full remaining public event stream and the faction-based outcome. There is no ghost-input phase in v1.

### 15.6 NPC model (deterministic heuristics)

Per-NPC suspicion score `S[i][j]` (i's view of j), updated by public events with fixed weights × persona multipliers:

| Event (about j, from i's view) | ΔS |
|---|---|
| Contradicted divination claims (two "seers" conflict) | +3 both claimants |
| j's accusation target was later revealed innocent | +1 |
| j defended a player later revealed wolf | +2 |
| j voted against a player later revealed innocent | +1 |
| j claims seer late (day ≥ 3 first claim) | +2 |
| confirmed-by-my-own-knowledge lie (I'm the real seer / fellow wolf) | +6 |
| j defended me while I'm suspected | −1 |
| random persona noise per day | ±0–1 (rng `npc:<id>`) |

Personas set integer thresholds (no fractional scaling): `aggressive` accuses at S ≥ 2 and may fake-claim; `cautious` accuses at S ≥ 5 and claims a day later; `follower` never initiates accusations and mirrors the current plurality vote intent. Wolves: know teammates. Alongside each NPC's private `S`, the engine maintains one shared **publicSuspicion** vector — the same weight table applied with neutral persona to public events only (the "town view" anyone can compute). Wolf kill targeting = highest `threat` (claimed seer > vocal accuser of a wolf > rng among non-wolves); wolf deflection: publicly accuse the non-wolf with the 2nd-highest publicSuspicion; never accuse/vote their partner unless partner publicSuspicion ≥ 8 ("bus" rule). Seer NPC: divines highest-suspicion unconfirmed player; claims + reports when (a) a wolf counter-claims, (b) it finds a wolf, or (c) day ≥ 3 (persona-adjusted). All NPC choices: argmax with rng tie-break, seat order as final tie-break.

### 15.7 Agent-facing views

`you.private`: your role; wolf → teammates + nightly kill choice; seer → divination results log. Public state: alive/dead + revealed roles, full structured speech log (acts + flavor), vote tallies per day, phase/round indicators, `expected` input kind. There is **no** channel exposing another player's private info; unit tests assert wolf identities absent from villager-view JSON (§19.4).

### 15.8 Spectator & rating

Active game → public view only (speech log, votes, deaths, revealed roles). After `finished` → full reveal (roles, night choices, suspicion trajectories for the post-mortem). Rating: single anchor 1200 (§10.2); practice (`agentRole` fixed) games unrated. Win = your faction wins (survival not required).

## 16. CLI & configuration

```
mcparcade                      # = serve: stdio MCP + spectator (default on)
mcparcade serve [--data-dir <p>] [--no-spectator] [--spectator-port <n>]
                [--http <port>] [--bind <addr>]        # HTTP transport mode (§17)
mcparcade validate-scenario <file.json> [--json]
mcparcade --version | --help
```

Env: `MCPARCADE_DATA_DIR`, `MCPARCADE_HTTP_TOKEN` (required for `--http`), `NO_COLOR`. Precedence: flag > env > default. stdout is sacred in stdio mode (§7.6): CLI human output for subcommands goes to stdout, but `serve` logs only to stderr.

## 17. Streamable HTTP transport (self-host, single-user)

- `mcparcade serve --http 8765` starts Streamable HTTP MCP at `/mcp` (SDK `StreamableHTTPServerTransport`) **instead of** stdio (one process, one transport), default bind `127.0.0.1` (choosing `--bind 0.0.0.0` prints a loud warning; intended use = behind your own TLS reverse proxy or on a trusted LAN/tailnet).
- Auth: static bearer token from `MCPARCADE_HTTP_TOKEN` (min 16 chars; server refuses to start with a shorter one; the server never generates or stores this secret — user supplies it). Constant-time compare; 401 without it. One user: every authenticated client shares the same profile/sessions (documented).
- Same Host/Origin validation approach as §11.3 when bound to loopback; behind a proxy the user sets `--allow-host <name>` (repeatable) to extend the allowlist.
- Spectator in HTTP mode: unchanged (separate port, own token).
- Rate limit: token-bucket 20 req/s burst 40, keyed by (source IP, MCP session id) (cheap in-memory; protects the box, not multi-tenant fairness). OAuth, multi-user, TLS termination: v2.

## 18. Security model

### 18.1 Assets & trust boundaries

Assets: the user's machine/files, the agent's context (prompt-injection target), game integrity (no cheating/leaks), profile data. Trust boundaries: (B1) MCP client ↔ tool facade · (B2) tool results → agent context · (B3) browser ↔ spectator · (B4) data dir files (incl. user scenarios) ↔ engine · (B5) HTTP transport ↔ network · (B6) npm supply chain ↔ user install.

### 18.2 Threat table (v1 mitigations are requirements)

| # | Threat | Boundary | Mitigation (design §) |
|---|---|---|---|
| T1 | Malicious/garbled tool args (traversal ids, huge payloads) | B1 | zod on every tool; id regex before path use; size caps (§7.4, §9.2) |
| T2 | Agent flooding (sessions, actions, log growth) | B1 | session cap, event/file caps, turn caps (§7.4) |
| T3 | Hidden-info leak (werewolf roles) | B1/B3 | per-player views computed server-side; leak unit tests (§15.7, §19.4) |
| T4 | DNS rebinding against spectator | B3 | loopback bind + Host/Origin allowlist (§11.3) |
| T5 | Local unauthorized spectator access (multi-user machine) | B3 | per-process bearer token, constant-time compare (§11.2) |
| T6 | XSS via game/scenario/flavor text in SPA | B3 | textContent-only rendering, CSP `default-src 'self'`, esc() helper (§11.6) |
| T7 | Malicious scenario file (zip-bomb-ish JSON, ref cycles) | B4 | full schema+referential validation, caps, skip-on-invalid (§14.3, §14.8) |
| T8 | Prompt injection from scenario prose into agent | B2 | documented trust note; no instructions-like metadata fields; author field rendered inert (§14.8) |
| T9 | Corrupt/hostile data-dir files crash server | B4 | quarantine-not-crash (§9.4) |
| T10 | Token/secret leakage in logs or files | B2/B3 | spectator token never persisted; URL token scrubbed from address bar; logs strip tokens; HTTP token env-only (§11.2, §17) |
| T11 | Unauthenticated/UNwanted network exposure in HTTP mode | B5 | loopback default, mandatory strong token, loud warning on 0.0.0.0, rate limit (§17) |
| T12 | Supply-chain (malicious transitive dep, typosquat) | B6 | 4-dep runtime budget, lockfile, `npm audit` + Dependabot + CodeQL in CI, publish with provenance, no postinstall scripts (§20) |
| T13 | Log injection (control chars in agent text) | B2 | strip control chars before stderr logging (§7.6) |
| T14 | stdout corruption breaking MCP framing | B1 | stderr-only logging rule + lint/test guard (§7.6, §19) |

### 18.3 Secure defaults summary

Loopback everywhere by default; tokens required on every non-stdio surface; no runtime outbound network I/O (assertable in tests: no `net`/`http` client imports outside spectator/transport server code); file perms 0600/0700; no secrets generated or stored except the ephemeral in-memory spectator token.

### 18.4 Input-validation chokepoints

(1) Tool facade: zod + size caps. (2) `storage/paths.ts`: the **only** module that turns ids into paths; validates against the ULID regex and rejects anything else — no other module may join paths into the data dir (enforced by review + a security test greping for `path.join(dataDir` outside `paths.ts`). (3) Scenario validator. (4) Spectator route params: same id regex; static file serving only from the bundled asset manifest (no directory reads).

## 19. Testing & validation strategy

### 19.1 Layers

| Layer | Tool | Gate |
|---|---|---|
| Unit (engines, rules, stores, elo, rng) | vitest | PR CI |
| Integration: full MCP round-trips via SDK `InMemoryTransport` (client ↔ server in-process) | vitest | PR CI |
| Determinism/replay invariants (§8) | vitest suite over every game | PR CI |
| Security tests (§19.4) | vitest | PR CI |
| Golden transcripts: scripted agent plays each game to completion | vitest fixtures | PR CI |
| Manual smoke: MCP Inspector + a real client (Claude Code) checklist | human | release |

### 19.2 Per-game minimums

Every game ships: rules-edge unit tests (listed in its issue), a golden transcript test (fixed seed, scripted actions, snapshot of final outcome + selected renders), and a determinism test (replay log ⇒ identical state).

### 19.3 Integration harness (issue 14)

`test/harness.ts` boots the real server with a temp data dir, connects an in-memory MCP client, and exposes `h.call(tool, args)`. All tool-level behavior (limits, errors, resume, record updates) is tested through it — not through internal APIs.

### 19.4 Security suite (issue 39)

Traversal ids (`../../x`, absolute, URL-encoded) on every id-taking tool/route → clean rejection; werewolf hidden-info leak assertions (serialize villager observation, assert no wolf ids/role strings); spectator: 401 without token, Host/Origin rejection matrix, CSP header presence, token absence from API bodies; oversized action/scenario rejection; corrupt session file → quarantine not crash; grep-guards (stdout writes, `path.join(dataDir` outside paths.ts).

### 19.5 Release checklist (issue 43)

`npm pack` audit (file whitelist), fresh-machine `npx mcparcade` smoke, Claude Code + Claude Desktop config walkthroughs from README verified, versioned CHANGELOG, provenance publish.

## 20. Packaging, release, repo infra

- Package `mcparcade@0.1.0`, `bin: {mcparcade: "dist/index.js"}`, `files: ["dist", "content", "README.md", "LICENSE"]`, `engines.node: ">=20"`. No postinstall. Provenance: `npm publish --provenance` from the release workflow (npm token is a repo secret **configured manually by the maintainer** — agents never handle it). The npm name is reserved early with a clearly marked `0.0.1` pre-release placeholder, published manually by the maintainer (squat defense; the name was verified available on 2026-07-11).
- CI (GitHub Actions): `ci.yml` = biome + tsc --noEmit + vitest on Node 20/22, ubuntu + macos. `codeql.yml` weekly + PR. Dependabot: npm + actions, weekly. Actions pinned to major versions; default `permissions: contents: read`.
- Branch protection / rulesets, secret scanning, push protection: repo settings applied manually by the maintainer (documented in issue 44 as a checklist; agents cannot and must not change org/repo security settings).
- Versioning: semver; tool-surface or storage-schema breaking changes bump major (pre-1.0: minor) and ship migrations (§9.2).

## 21. Known unknowns (may spawn issues during implementation)

| # | Unknown | Trigger to resolve |
|---|---|---|
| U1 | tsshogi edge-rule coverage (uchifuzume, sennichite detail) | Issue 24 acceptance tests; fallback = in-adapter checks or shogi.js swap |
| U2 | Shogi NPC strength vs node caps (is "hard" fun and ≤ 5 s?) | Issue 25 self-play calibration; adjust caps/eval constants |
| U3 | Sudoku generator yield/quality for "hard" | Issue 18; fallback = curate from generator output manually |
| U4 | Werewolf NPC believability (heuristics may feel robotic) | Issue 35 playtest transcript review; tune weights (constants file) |
| U5 | `fs.watch` reliability across platforms for cross-process spectating | Issue 20; fallback = 2 s polling |
| U6 | MCP SDK minor-version API drift before implementation starts | Issue 01 pins the verified version; adapter layer confines changes |
| U7 | claude.ai custom-connector compatibility for self-hosted HTTP (auth expectations) | Issue 38 manual verification; may add `--allow-host` docs |
| U8 | Token-cost ergonomics of observations in long games (shogi 100+ plies) | Golden transcripts measure render sizes; tune render verbosity |

## 22. Glossary

**Session** — one playthrough of one game. **Observation** — the agent-visible view. **NPC** — server-controlled player (non-LLM). **Anchor rating** — fixed Elo assigned to an NPC difficulty. **Replay** — finalized session record sufficient for deterministic playback. **Spectator** — the read-only local web UI. **SFEN/USI** — shogi position/move notations. **Speech act** — structured werewolf utterance component.
