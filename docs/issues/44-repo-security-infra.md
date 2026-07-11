# Title

Repository security infrastructure: CodeQL, Dependabot, actions pinning, manual-settings checklist

## Summary

Add the security automation workflows per DESIGN.md §20 — CodeQL scanning, Dependabot (npm + actions), workflow-permissions hardening — plus a documented checklist of repo settings the maintainer applies manually (rulesets, secret scanning, push protection).

## Context

The repo becomes public OSS; supply-chain and repo-surface hardening (T12) must be in place *before* publicity, not after. Agents must not change org/repo security settings; those are checklist items for the human maintainer.

## Scope

- `.github/workflows/codeql.yml`: JavaScript/TypeScript, on PR + weekly cron; default CodeQL config; `permissions: {actions: read, contents: read, security-events: write}`.
- `.github/dependabot.yml`: npm (weekly, grouped minor/patch, separate majors) + github-actions (weekly).
- `npm audit` in CI (T12 requires audit + Dependabot + CodeQL): add an `audit` job to `ci.yml` running `npm audit --omit=dev --audit-level=high` (blocking on high/critical).
- Audit pass over ALL workflows existing at this point (02 ci, 43 release, codeql — issue 43 is a dependency so the release workflow exists): explicit least-privilege `permissions` per workflow, no `pull_request_target`, no third-party actions beyond `actions/*` and `github/codeql-action` (pinned to major).
- `docs/guides/repo-settings-checklist.md` (manual, maintainer-executed): default-branch ruleset (PR required, no force-push, no direct push), secret scanning + push protection ON, private vulnerability reporting ON, Actions "read repository contents" default token policy, tag protection `v*`, require CI + CodeQL checks before merge.
- `.github/PULL_REQUEST_TEMPLATE.md` + issue templates referencing the docs-first rule (issues derive from `docs/issues/`).

## Detailed Requirements

1. CodeQL must pass on the current codebase; any findings triaged in the PR (fix or dismiss-with-reason inline).
2. Dependabot grouping keeps noise low: one weekly PR for npm minors/patches; majors individual.
3. The checklist has checkboxes with exact GitHub UI paths (Settings → Rules → …), `gh api` equivalents where they exist, and concrete values: default-branch ruleset = require a PR before merging (0 required approvals — solo maintainer), require status checks `ci` and `CodeQL` to pass, block force pushes, restrict deletions, no bypass actors; secret scanning + push protection ON; private vulnerability reporting ON; Actions workflow permissions default "Read repository contents"; tag protection ruleset for `v*`. All items marked "maintainer-manual".
4. Templates: PR template asks "which docs/issues/NN file does this implement? checklist: tests added, docs updated"; bug/feature issue templates route feature *design* changes to DESIGN.md discussion first.
5. No workflow may have `write` beyond what it strictly needs (`security-events: write` for CodeQL; `id-token: write` only in release).

## Acceptance Criteria

- [ ] CodeQL runs green on this issue's PR (run link); a post-merge checklist item in the PR description records the first default-branch run link (verified by the merger).
- [ ] `npm audit` job present in `ci.yml` and green.
- [ ] Dependabot config recognized by GitHub: the repo's Insights → Dependency graph → Dependabot tab lists both ecosystems as active (screenshot or `gh api /repos/{owner}/{repo}/dependabot/alerts` accessibility check in the PR), or the first grouped PR link.
- [ ] Workflow-permissions audit table (workflow → permissions → justification) included in the PR description.
- [ ] Checklist doc complete **and executed by the maintainer with evidence attached to the PR** (screenshots or `gh api` output per item) — the repository is already public, so completion of this issue requires the settings to actually be in place, not deferred.
- [ ] Templates exist at `.github/PULL_REQUEST_TEMPLATE.md` and `.github/ISSUE_TEMPLATE/*.md|yml`, and a screenshot shows GitHub's new-issue/new-PR pickers rendering them with the expected fields.

## Validation

- Workflow run links + maintainer checklist confirmation recorded in the PR.

## Dependencies

- 01, 02, 43 (the release workflow must exist to be audited).

## Non-goals

- Changing org/repo settings programmatically (maintainer-manual by policy), OpenSSF Scorecard badge chasing (may adopt later), SBOM generation (v2 consideration).

## Design References

- DESIGN.md §20, §18.2 (T12)
