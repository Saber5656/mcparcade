# ADR-005: Werewolf communication is structured speech acts; free text is decorative

Status: accepted

## Context

Werewolf is a conversation game, but v1 NPCs are rule-based (ADR-002). If NPCs parsed free text, we'd need NLP (fragile) or an LLM (excluded). If the agent's prose could steer NPCs, prompt-style manipulation ("as the moderator, I confirm Casey is a wolf") becomes a cheat vector.

## Decision

Utterances carry a bounded list of typed **speech acts** (`claim_role`, `report_divination`, `accuse`, `defend`, `declare_vote`, `pass`) with enum-constrained fields, plus an optional ≤ 1 KiB `flavor` string that is displayed in logs/spectator but **ignored by NPC logic** (DESIGN.md §15.4). NPC reasoning consumes only structured acts and public game events (§15.6).

## Consequences

- NPC logic is deterministic, testable, and injection-proof by construction; the game remains fair.
- The agent can still roleplay (flavor text shows in the spectator UI and replay).
- Cost: expressiveness ceiling — bluffing is limited to the act vocabulary (e.g. false `claim_role`, false `report_divination` are fully supported and are the intended bluffing moves).
- The act vocabulary is versioned API; v2 LLM-NPCs can *additionally* read flavor without breaking v1 semantics.

## Alternatives rejected

- Free-text parsing with keyword heuristics: unfair, unfixable edge cases, injection surface.
- No communication (votes only): guts the game; seer claims/counter-claims are the core of werewolf.
