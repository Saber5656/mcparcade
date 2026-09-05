# ADR-002: Built-in engines / rule-based NPCs; the server never calls an LLM

Status: accepted (user decision, 2026-07-10)

## Context

Shogi needs an opponent; werewolf needs six other players. Options: built-in deterministic engines/heuristics, LLM-driven NPCs (server calls a model API), or multi-agent lobbies. The product thesis is "your agent becomes the AI that plays" — the server is the game world, not another AI.

## Decision

v1 NPCs are **in-process, deterministic, non-LLM**: a node-capped negamax engine for shogi (DESIGN.md §13.5) and a weighted-heuristic suspicion model with structured speech for werewolf (§15.6). The server makes **zero outbound network requests** and handles **zero API keys**.

## Consequences

- Zero-friction install (no keys), zero marginal cost, offline play, reproducible games (determinism §8), drastically smaller threat model (no credential handling, no egress).
- NPC conversation/strength ceilings are real: werewolf NPCs are believable only within their structured-speech world; shogi tops out at "club beginner". Accepted for v1; U2/U4 track tuning.
- The `GameDefinition.npcAct` seam plus per-NPC rng streams keep the door open for v2 LLM-NPC adapters (user-supplied key) and agent-vs-agent lobbies without reworking games.

## Alternatives rejected

- LLM NPCs in v1: forces API-key UX and secret handling, non-determinism breaks replays/ratings, adds prompt-injection surface between players, per-game cost.
- External engines (USI/Stockfish-style): arbitrary-binary execution surface and the #1 setup-failure mode in prior art (docs/research/prior-art-game-mcp.md).
