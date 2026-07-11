# Research: Prior Art — Game MCP Servers

Date: 2026-07-11
Purpose: position mcparcade against existing "AI plays games over MCP" projects and steal proven lessons. This survey shaped ADR-001 (single server, uniform tool surface) and DESIGN.md §5.

## Landscape (July 2026)

| Project | What it is | Relevant takeaway |
|---|---|---|
| ChessAgine MCP (jalpp) | Chess awareness server: board state, Stockfish analysis, Lichess/opening data | Rich *analysis* tooling, but the AI is an analyst, not a *player with a record*. Tool surface is chess-specific. |
| alexandreroman/mcp-chess | Play chess vs. the LLM over MCP | Proves the core loop (observe → move → engine replies) works in chat clients. Single game, no persistence. |
| pab1it0/chess-mcp | Chess.com published-data API wrapper | Data access, not gameplay. |
| Shogi USI-bridge MCP servers | Position analysis via external USI engines over HTTP | Depends on user-installed engine binaries — exactly the setup friction and security surface we avoid in v1 (ADR-002). |
| fritzprix/role-playing-mcp-server | LLM-narrated RPG session manager | Free-form narration; no deterministic rules engine, no fairness guarantees. |
| Civilization VI MCP | Bridge into a running commercial game | Shows appetite for "my agent plays games", but requires the game installed; not self-contained. |
| Glama "Games & Gamification" category | Dozens of single-game or gimmick servers | Fragmented: one server per game, no shared session/record model. |

## Gap analysis → mcparcade differentiation

1. **One server, many games, one uniform tool surface** (`list_games` / `describe_game` / `start_game` / `observe` / `act` / …). Prior art is one-server-per-game, which floods client configs and tool lists.
2. **The agent accumulates a persistent identity**: local match records, Elo-style ratings vs. calibrated NPC anchors, replays. No surveyed project persists "how good is my agent at games" locally.
3. **Secretless and offline**: built-in engines and rule-based NPCs; zero API keys, zero outbound network traffic at runtime. USI-bridge and analysis servers require engine installs or external APIs.
4. **Hidden-information games done honestly** (werewolf): per-player views enforced server-side, not by prompt convention. No surveyed MCP server does social deduction with information hygiene.
5. **Human spectator surface**: a local read-only web UI with live updates and replay playback. Prior art renders boards into chat only.

## Lessons adopted

- Chat-rendered boards must be *monospace-safe and compact*; chess servers that dump FEN + ASCII + commentary in one blob waste tokens. → Observation format separates `render` (text) from `state` (structured JSON) and makes verbose parts opt-in (DESIGN.md §6.4).
- Always return legal actions (or a precise action schema) with the observation; retry loops from invalid moves dominate transcripts of naive game servers. → `legalActions` is a first-class observation field (DESIGN.md §6.4, §7.3).
- External engine processes (Stockfish/USI bridges) are the #1 setup-failure and an arbitrary-binary-execution surface. → v1 ships no external engine support; deferred to v2 with an explicit ADR gate.

## Sources

- https://glama.ai/mcp/categories/games-and-gamification
- https://github.com/jalpp/chessagine-mcp
- https://github.com/alexandreroman/mcp-chess
- https://github.com/pab1it0/chess-mcp
- https://github.com/fritzprix/role-playing-mcp-server
