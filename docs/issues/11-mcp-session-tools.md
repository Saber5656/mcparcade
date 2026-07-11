# Title

MCP session tools: `start_game`, `observe`, `act`

## Summary

Register the three gameplay tools on the server from issue 10, as thin validated facades over the runtime (issue 09), matching DESIGN.md §5.3–§5.5 exactly.

## Context

These are the tools an agent uses hundreds of times per game; their input validation, error ergonomics (§7.3), and response shapes decide whether LLM play is smooth or a retry mess.

## Scope

- `start_game {gameId, options?}` → `{sessionId, observation, spectatorUrl}`.
- `observe {sessionId, verbose?}` → `{observation}`.
- `act {sessionId, action}` → `{accepted, events, observation, outcome?}` (plus `error` when rejected).
- Facade-level guards: action payload ≤ 8 KiB (serialized), sessionId regex pre-check before hitting storage.

## Detailed Requirements

1. Input schemas: `gameId: z.string().min(1).max(32)`, `sessionId` validated with `isValidId("session", …)` at the facade (fast fail, §18.4), `action: z.record(z.unknown())`, `options: z.record(z.unknown()).optional()` — deep validation (per-game options schema, the universal `seed` regex, per-game action schema) belongs to the runtime (issue 09), not the facade. The facade's job: shape, ids, size.
2. Responses: `structuredContent` matches §5.3–§5.5 shapes; `text` renders the observation's `render` field plus a one-line status (and on rejection: the error message + hint). The `render` is included in `text` on every `act`/`observe` so chat clients display the board without extra calls.
3. Rejected `act` (domain-level) returns `accepted: false` + `error{code,message,hint}` + unchanged observation — as a *successful* tool call (§7.5). The §7.3 ergonomics surface verbatim from the runtime: rejections on list-mode games include up to 10 sample legal actions inside `error.hint`/`structuredContent.error.samples`, and after 10 consecutive rejections the full legal-action list (or schema summary) is appended.
4. `spectatorUrl` from the injected provider (issue 10's `spectatorInfo`); include it in `start_game` responses only (per §5.9 it also appears in `list_games`/`list_sessions`).
5. Oversized `action` (>8 KiB) rejected at the facade with `E_INVALID_ACTION` and hint "action too large" — never reaches the runtime.
6. Tool descriptions (LLM-facing): `act` description must state: "If your action is rejected, read error.hint and check observation.legalActions; rejected actions never change the game."

## Acceptance Criteria

- [ ] Stub-game full loop via in-memory client: start → observe → act (accept) → act (reject) → resign; every response validates against the §5 zod response schemas (write them in `test/` as executable spec).
- [ ] Malformed sessionIds (traversal strings, wrong prefix) rejected at the facade with exactly `error.code == "E_SESSION_NOT_FOUND"` and `hint` containing `sess_<26-character-ULID>`; storage untouched (spy).
- [ ] 8 KiB+1 action rejected with `E_INVALID_ACTION`; 8 KiB−1 passes the facade.
- [ ] Rejection ergonomics: stub list-mode game rejection carries ≤ 10 samples; the 11th consecutive rejection carries the full list (integration through the `act` tool).
- [ ] `verbose: true` flips `legalActions.mode` to `list` on the stub game configured with 100 actions.
- [ ] Text block contains the render on both `observe` and accepted `act`; response invariants hold on every call in the suite (`text` ≤ 16 KiB, `structuredContent` ≤ 64 KiB — asserted via the issue-10 registrar guards).

## Validation

- `test/integration/session-tools.test.ts` via the SDK in-memory pair (migrates onto issue 14's harness when it lands).

## Dependencies

- 09, 10.

## Non-goals

- list/record tools (12), transports (13), any real game.

## Design References

- DESIGN.md §5.3–§5.5, §5.9, §7.3, §7.5, §18.2 (T1, T2), §18.4 (chokepoint 1)
