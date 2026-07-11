# Title

MCP server bootstrap and catalog tools: `list_games`, `describe_game`

## Summary

Implement `src/server/tools.ts` bootstrap (McpServer instance, shared tool-result helpers, error mapping) plus the two catalog tools per DESIGN.md §5.1–§5.2.

## Context

First MCP-visible surface. Establishes the conventions every other tool follows: zod input schemas, dual `text` + `structuredContent` responses, and §7.5 error objects returned as readable tool results (never thrown as protocol errors for domain failures).

## Scope

- `createMcpServer({runtime, registry, spectatorInfo}) → McpServer`: creates the server and a shared `registerArcadeTool(name, schema, handler)` registrar that applies the common conventions (validation, size guards, error mapping, dual text+structured output). **This issue registers only `list_games` and `describe_game`; the server exposes exactly the tools registered so far** — issues 11/12 call the same registrar to add theirs.
- Shared helpers: `ok(text, structured)`, `err(McparcadeError)` (structured `{error:{code,message,hint}}` + text line), and size guards: request arguments ≤ 16 KiB serialized (reject `E_INVALID_ACTION`-style with a "request too large" hint); response invariants `text ≤ 16 KiB`, `structuredContent ≤ 64 KiB` asserted in the registrar (throwing `E_INTERNAL` in dev/test if a handler exceeds them — §5 preamble).
- `list_games`: catalog from the registry + `spectatorUrl` (nullable; wired later by issue 19 — inject a `spectatorInfo: {url: string|null}` provider now).
- `describe_game`: rules/actions/options/examples/tips assembled from `GameDefinition` fields (`rulesDoc` + generated option/action documentation from the zod schemas; rating anchors from issue 08 appear in shogi/werewolf docs — data-driven, not hard-coded here).

## Detailed Requirements

1. Tool registration via SDK `registerTool` with zod raw shapes; descriptions written for LLM consumption: `list_games` description ends with "call describe_game before starting a game you haven't played in this conversation".
2. `structuredContent` mirrors DESIGN.md §5.1/§5.2 field names exactly; `text` block is a compact human-readable digest (≤ 40 lines).
3. Zod→docs generation: implement `describeSchema(zodType) → string` producing the `options`/`actions` documentation (discriminated unions listed per `type` with field names/enums/ranges). Keep it simple and deterministic; snapshot-tested.
4. Unknown `gameId` → `E_GAME_NOT_FOUND` with hint listing valid ids.
5. Domain errors are returned as successful tool calls with the `error` object (per §7.5); only zod input-shape failures surface as SDK-level invalid-params.
6. No transport code here (stdio wiring is issue 13); the module exports a pure factory testable with `InMemoryTransport`.

## Acceptance Criteria

- [ ] With two stub games registered, `list_games` returns both with all §5.1 fields; `spectatorUrl` null when provider says so.
- [ ] `describe_game("stub")` includes rulesDoc text, generated action docs listing every action `type`, options docs including universal `seed`, ≥ 2 examples.
- [ ] `describe_game("nope")` → tool result with `isError` **not** set (successful call per §7.5), structured `error.code == "E_GAME_NOT_FOUND"` with hint.
- [ ] Size guards: a 16 KiB+1 argument payload is rejected with the "request too large" hint; a test handler returning an oversized response trips the registrar assertion.
- [ ] Snapshot test of `describeSchema` on a representative discriminated union (stable across runs).
- [ ] All responses carry both `text` and `structuredContent`.

## Validation

- `test/integration/catalog.test.ts` via SDK InMemory client pair (pre-harness; issue 14 will refactor onto the shared harness).

## Dependencies

- 03, 08 (anchor constants surfaced in `describe_game` docs).

## Non-goals

- Session tools (11), record tools (12), transports (13), spectator URL provisioning (19).

## Design References

- DESIGN.md §5.1–§5.2, §5.9, §7.5; docs/research/stack-and-mcp-sdk.md
