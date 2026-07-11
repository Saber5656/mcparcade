# Research: Runtime Stack and MCP SDK

Date: 2026-07-11
Status: verified facts recorded below; re-verify pinned versions at implementation time.

## Decisions this research supports

- TypeScript / Node.js single-package server distributed via `npx mcparcade` (user-confirmed stack).
- Official MCP TypeScript SDK as the only protocol dependency.
- stdio transport first; Streamable HTTP self-host mode behind a flag (ADR-003).

## Verified facts (2026-07-11)

| Fact | Value | How verified |
|---|---|---|
| npm package name `mcparcade` | **available** (404 on registry) | `npm view mcparcade` → E404 |
| `@modelcontextprotocol/sdk` latest | **1.29.0** | `npm view @modelcontextprotocol/sdk version` |
| `tsshogi` latest | 2.3.4, MIT | `npm view tsshogi version license` |
| `shogiops` latest | 0.21.0, GPL-3.0-or-later | `npm view shogiops version license` |
| `shogi.js` latest | 5.5.0, MIT | `npm view shogi.js version license` |

Action item captured in issue 43 (packaging): reserve/publish the `mcparcade` npm name early to avoid squatting before the public release.

## MCP TypeScript SDK usage notes

- Server API: `McpServer` with `registerTool(name, { description, inputSchema }, handler)` where `inputSchema` is a Zod raw shape. Tool results are returned as `content` blocks; we return a human-readable text block plus a `structuredContent` JSON object for machine consumption.
- Transports: `StdioServerTransport` (v1 default) and `StreamableHTTPServerTransport` (self-host mode). Both wrap the same `McpServer` instance; the game runtime must not know which transport is active.
- stdio discipline: a stdio MCP server MUST NOT write anything except JSON-RPC to stdout. All logging goes to stderr. This is a hard requirement carried into DESIGN.md §7.6 and the scaffold issue.
- Pinning: depend on `@modelcontextprotocol/sdk` `^1.29.0`. The SDK is pre-2.0 and minor versions have historically adjusted server APIs; implementation issues instruct the implementer to check the changelog when installing.
- Capability surface used in v1: **tools only**. Resources/prompts/sampling are deliberately not used (widest client compatibility; claude.ai, Claude Code, Claude Desktop, Codex CLI, Cursor all support tools).

## Runtime / toolchain choices (for the scaffold issue)

| Concern | Choice | Rationale |
|---|---|---|
| Node version | `>=20` (`engines.node`) | Active LTS floor; `fs.watch` recursive on macOS/Linux, stable `node:test`-era APIs; we still use vitest |
| Module system | ESM only (`"type": "module"`) | SDK is ESM-first; avoids dual-build complexity |
| Build | `tsup` (esbuild) → `dist/` | Single-file-ish fast builds for a `bin` entry |
| Tests | `vitest` | Fast TS-native, snapshot support for renders |
| Lint/format | `Biome` | One tool, zero-config, fast; no plugin sprawl |
| Schema validation | `zod` | Required by MCP SDK tool registration anyway; reused for storage/scenario validation |
| CLI parsing | `commander` | Small, ubiquitous |
| IDs | ULID, implemented in-repo (~30 lines) | Avoids a dependency for something trivial; sortable ids |

Runtime dependency budget (hard cap, enforced in review): `@modelcontextprotocol/sdk`, `zod`, `commander`, `tsshogi`. Everything else (SSE, HTTP, file watching, RNG, ULID, Elo) is implemented on Node built-ins. Rationale: OSS supply-chain surface minimization (DESIGN.md §18).

## Sources

- npm registry lookups executed locally on 2026-07-11 (values above).
- MCP TypeScript SDK: https://github.com/modelcontextprotocol/typescript-sdk
- MCP specification (transports, tools): https://modelcontextprotocol.io/specification
