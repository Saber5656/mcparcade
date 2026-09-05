# Title

npm packaging and release: provenance workflow, version 0.1.0 checklist

## Summary

Finalize the publishable package per DESIGN.md §20: `files` whitelist verification, `bin` smoke, CHANGELOG, a tag-triggered GitHub Actions release workflow publishing with `--provenance`, and the executable 0.1.0 release checklist (§19.5).

## Context

The last gate before public availability. The npm name `mcparcade` was verified available (research doc); early name reservation is part of this issue. Secrets (npm token) are configured manually by the maintainer — the workflow only consumes them.

## Scope

- `package.json` release fields (exact values): `description: "A mini-game arcade MCP server — turn your AI agent into an AI that plays games"`, `keywords: ["mcp", "model-context-protocol", "games", "shogi", "werewolf", "ai-agent"]`, `repository: {"type": "git", "url": "git+https://github.com/Saber5656/mcparcade.git"}`, `homepage: "https://github.com/Saber5656/mcparcade#readme"`, `bugs: "https://github.com/Saber5656/mcparcade/issues"`, `author: "Saber5656"`. This is a **bin-only** package (no public JS API in v1): set `"exports": {"./package.json": "./package.json"}` to lock internals while keeping the bin functional (`bin` resolution is independent of `exports` — verify against current Node/npm behavior at implementation time and record the check in the PR).
- `.github/workflows/release.yml`: two triggers, same steps — (a) push of tag `v*` → real publish; (b) `workflow_dispatch` with required input `dryRun: true` → identical pipeline but `npm publish --dry-run` and the tag/version-match check skipped (this is the sanctioned dry-run mechanism; the tag path never accepts non-tag refs). Steps: checkout, setup-node (Node 22, `registry-url`), `npm ci`, full test suite, publish with `--provenance --access public`; `permissions: {contents: read, id-token: write}`; environment `release` (maintainer-approved).
- `CHANGELOG.md` (Keep a Changelog format), 0.1.0 entry drafted from the issue list.
- Name reservation (sanctioned by DESIGN.md §20): the maintainer manually publishes a clearly marked `0.0.1` placeholder (README + bin printing "pre-release") as soon as this issue starts — squat defense; documented as a maintainer-manual step.
- The §19.5 checklist as `docs/guides/release-checklist.md` (fresh-machine npx smoke, client walkthroughs, spectator pass, `npm pack` audit, version bump script `npm version`).

## Detailed Requirements

1. `npm pack` audit is a CI test: pack in a temp dir and assert the tarball's sorted path list equals exactly `package.json` + `README.md` + `LICENSE` + everything under `dist/` and `content/` (npm auto-includes the first three regardless of `files`; the assertion snapshot is the committed expected manifest, regenerated deliberately), no `src/`/`test/`/`docs/`/dotfiles, total unpacked size < 5 MB. The same test asserts `package.json` in the tarball has **no** `preinstall`/`install`/`postinstall`/`prepare` scripts (§20/T12).
2. Post-pack smoke test in CI: install the packed tarball into a temp project, run `mcparcade --version` and a 5-second stdio handshake (`initialize` + `tools/list`) against the installed bin.
3. Tag-triggered runs verify `package.json` version equals the tag (checked step; mismatch fails). Supply-chain step: `npm audit --omit=dev --audit-level=high` runs in the release pipeline and fails the release on high/critical findings (T12).
4. Provenance requires GitHub-hosted runners + npm ≥ 9.5: pin the workflow's node to 22 and document.
5. Maintainer-manual steps documented (not automated): creating the npm token/trusted-publisher config, GitHub `release` environment protection, the 0.0.1 reservation publish. Agents never handle these secrets (repo rule).
6. License/NOTICE hygiene: verify `tsshogi` MIT attribution requirements (bundle its LICENSE text in `dist/` if bundling its code via tsup — check tsup output; if tsshogi is bundled, emit `dist/THIRD_PARTY_LICENSES.txt` generated at build).

## Acceptance Criteria

- [ ] CI pack-audit (manifest + no-install-scripts) + tarball smoke tests green.
- [ ] Dry-run release via `workflow_dispatch dryRun=true` executes all steps including `npm publish --dry-run` (run link in the PR).
- [ ] `THIRD_PARTY_LICENSES.txt` present in the tarball when bundling (or dependency shipped unbundled — decision recorded).
- [ ] Release checklist doc complete and every item executable by a non-author.
- [ ] 0.0.1 reservation published (or a documented decision to skip with the risk accepted by the maintainer).
- [ ] Tag/version mismatch fails the workflow (tested with a scratch mismatch tag on a fork or scratch repo; run link in the PR).

## Validation

- CI tests above; dry-run workflow link in the PR; checklist walkthrough on a second machine recorded.

## Dependencies

- 02, 39, 40, 41 (the checklist references shipped docs and green suites).

## Non-goals

- Actually cutting 0.1.0 (maintainer decision; merge ≠ release per repo policy), homebrew/deb packaging, auto-changelog generation.

## Design References

- DESIGN.md §20 (incl. the sanctioned 0.0.1 name reservation), §19.5, §18.2 (T12); docs/research/stack-and-mcp-sdk.md (name availability)
