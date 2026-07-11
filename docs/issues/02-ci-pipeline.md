# Title

CI pipeline: lint, typecheck, tests on every PR

## Summary

Add a GitHub Actions workflow that runs Biome, `tsc --noEmit`, and vitest on Node 20 and 22, on ubuntu and macos, for every pull request and push to `main`.

## Context

All later issues gate on "CI green". Security-oriented workflows (CodeQL, Dependabot) are issue 44; this issue is only the correctness pipeline.

## Scope

- `.github/workflows/ci.yml`.
- A status badge in README is out of scope (README is rewritten in issue 41).

## Detailed Requirements

1. Triggers: `pull_request` (all branches) and `push` to `main`.
2. Job matrix: `node: [20, 22]` × `os: [ubuntu-latest, macos-latest]`.
3. Steps: checkout → setup-node with npm cache → `npm ci` → `npm run lint` → `npm run typecheck` → `npm test -- --run`.
4. Top-level `permissions: contents: read` (least privilege).
5. Actions pinned at least to major version tags (`actions/checkout@v4`, `actions/setup-node@v4`); no third-party actions.
6. `concurrency` group per ref with `cancel-in-progress: true`.
7. Total runtime target < 5 minutes per job at current repo size.

## Acceptance Criteria

- [ ] Workflow passes on a PR touching only docs (proving cache + install work).
- [ ] A deliberately failing test on a scratch branch turns the PR red (verified once, then reverted).
- [ ] `permissions` block present; no `write` permission anywhere in the workflow.
- [ ] Matrix runs all four combinations.

## Validation

- Manual: open a scratch PR demonstrating green run and red run (screenshots or run links in the PR description).

## Dependencies

- 01 (scripts must exist).

## Non-goals

- CodeQL, Dependabot, release/publish workflows (issues 43, 44).

## Design References

- DESIGN.md §20 (CI requirements)
