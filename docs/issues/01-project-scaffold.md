# Title

Project scaffold: package, TypeScript/ESM, build, test, lint toolchain

## Summary

Create the buildable, testable, lintable skeleton of the `mcparcade` npm package exactly as specified in DESIGN.md §4.1 and docs/research/stack-and-mcp-sdk.md, with placeholder modules so every later issue lands into a fixed layout.

## Context

The repository currently contains only `README.md`. Every other issue assumes this layout, toolchain, and dependency budget. Toolchain choices were fixed by research (stack-and-mcp-sdk.md): Node ≥ 20, ESM-only, tsup, vitest, Biome, zod, commander.

## Scope

- `package.json`, `package-lock.json` (committed), `tsconfig.json`, `biome.json`, `vitest.config.ts`, `tsup.config.ts`, `.gitignore`, `.editorconfig`, `LICENSE` (MIT, copyright holder "mcparcade contributors").
- Directory skeleton of DESIGN.md §4.1: `src/{server,core,storage,rating,spectator,games}`, repo-root `content/` (bundled data), `test/{unit,integration,security,determinism}`, `tools/` — with `index.ts` placeholder exports where needed to keep `tsc` green.
- npm scripts: `build`, `dev` (tsx or tsup watch), `test`, `lint`, `format`, `typecheck`.
- One trivial passing test to prove the pipeline.

## Detailed Requirements

1. `package.json`: `"name": "mcparcade"`, `"version": "0.0.0"`, `"type": "module"`, `"license": "MIT"`, `"engines": {"node": ">=20"}`, `"bin": {"mcparcade": "dist/index.js"}`, `"files": ["dist", "content", "README.md", "LICENSE"]`.
2. Runtime dependencies — exactly these four, nothing else: `@modelcontextprotocol/sdk` (`^1.29.0`; check the changelog if a newer 1.x exists at install time and record the installed version in the lockfile), `zod`, `commander`, `tsshogi` (`^2.3.4`). Dev dependencies: `typescript`, `tsup`, `vitest`, `@biomejs/biome`, `tsx`, `@types/node`.
3. `tsconfig.json`: `"strict": true`, `"module": "NodeNext"`, `"moduleResolution": "NodeNext"`, `"target": "ES2022"`, `"noUncheckedIndexedAccess": true`.
4. tsup: entry `src/index.ts`, format `esm`, `banner: {js: "#!/usr/bin/env node"}` — verify the built `dist/index.js` is executable via `npx .`.
5. `src/index.ts`: commander program with `--version` and a stub `serve` default command that prints "serve: not implemented yet (see issue 13)" **to stderr** and exits 1. This intentionally deviates from DESIGN.md §16 until issue 13 replaces the stub; the stub behavior is itself an acceptance criterion below.
6. Biome: recommended rules; no custom disables except line width 110.
7. `.gitignore`: `node_modules/`, `dist/`, `*.corrupt-*`, coverage output.
8. No `postinstall`/`prepare` scripts that execute code beyond `build` on publish (`prepublishOnly: npm run build && npm test` is allowed).

## Acceptance Criteria

- [ ] `package-lock.json` is committed; `npm ci && npm run build && npm test && npm run lint && npm run typecheck` all succeed on a clean checkout, Node 20 and 22.
- [ ] `node dist/index.js --version` prints the package version and exits 0; `node dist/index.js serve` prints the not-implemented stub message to stderr and exits 1.
- [ ] `npm pack --dry-run` lists only `dist/`, `content/`, `README.md`, `LICENSE`, `package.json`.
- [ ] Runtime dependency list is exactly the four packages above.
- [ ] Directory skeleton matches DESIGN.md §4.1 (empty dirs carry a `.gitkeep` or placeholder module).

## Validation

- CI-runnable: the commands in the first acceptance criterion.
- Manual: `npx . --version` from the repo root resolves and runs the bin (prints the version, exit 0) without a global install.

## Dependencies

None (first issue).

## Non-goals

- No MCP wiring, no game code, no CI workflow files (issue 02), no README rewrite (issue 41).

## Design References

- DESIGN.md §4.1 (layout), §16 (CLI shape), §20 (packaging constraints)
- docs/research/stack-and-mcp-sdk.md (versions, dependency budget)
