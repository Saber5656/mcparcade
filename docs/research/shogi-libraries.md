# Research: Shogi Rules Libraries for TypeScript

Date: 2026-07-11
Decision: use **tsshogi** as the shogi rules/notation foundation. Recorded here because the choice materially affects issue 24 (shogi rules adapter) and the product license (MIT).

## Requirements

The shogi game plugin (DESIGN.md §13) needs, in order of importance:

1. Legal move generation and validation for standard shogi, including drops.
2. Special-rule handling: nifu (二歩), last-rank drop restrictions, uchifuzume (打ち歩詰め), sennichite (千日手) incl. perpetual-check loss, checkmate/stalemate detection.
3. SFEN import/export and USI move notation (the agent-facing action format).
4. MIT-compatible license (product is MIT).
5. Active maintenance, TypeScript types.

## Candidates

| Library | Version | License | Notes |
|---|---|---|---|
| **tsshogi** (sunfish-shogi) | 2.3.4 | MIT | Extracted from/used by ShogiHome (formerly Electron Shogi). Wide format support: SFEN/USI, KIF, KI2, CSA, JKF. Position/Record model, legal move checks. Actively maintained. |
| shogiops (WandererXII) | 0.21.0 | **GPL-3.0-or-later** | Technically strong (lishogi ecosystem): legal move/drop generation, setup validation, SFEN/USI/KIF/CSA. **Excluded**: GPL is incompatible with the MIT distribution goal. |
| shogi.js (na2hiro) | 5.5.0 | MIT | Simple board/piece model with move legality basics. Fallback option; fewer edge rules and formats than tsshogi. |

## Decision

- Primary: **tsshogi 2.3.x** wrapped behind our own `ShogiRules` adapter interface so the library never leaks into game-plugin code.
- Fallback (only if validation fails): `shogi.js` + in-repo implementations of the missing edge rules.
- `shogiops` must not be added as a dependency in any form (license). Reading its documentation/tests for *ideas* is fine; copying code is not.

## Risk: edge-rule coverage is unverified

Exactly which of nifu / uchifuzume / sennichite / jishogi (impasse) tsshogi enforces out of the box is **not verified yet** and is a known unknown (ISSUE_PLAN.md). Issue 24 therefore:

- ships an acceptance test suite with named SFEN positions for each edge rule (nifu, last-rank drops, uchifuzume, sennichite repetition draw, perpetual check), and
- requires the adapter to implement any check the library lacks, rather than weakening the tests.

Jishogi / impasse (27-point declaration) is explicitly **out of v1 scope** (DESIGN.md §13.7): games hitting the turn cap end as `aborted_turn_limit`. Rationale: extremely rare vs. our weak NPC, complex to specify, and safe to defer.

## Sources

- https://github.com/sunfish-shogi/tsshogi
- https://github.com/WandererXII/shogiops (license: GPL-3.0-or-later)
- https://github.com/na2hiro/Shogi.js
- npm registry license/version lookups executed locally on 2026-07-11.
