# Title

Escape scenario authoring guide

## Summary

Write `docs/guides/authoring-escape-scenarios.md`: the complete, example-driven guide for P3 content authors covering the §14.2 schema, design patterns, validation workflow, and publishing etiquette.

## Context

Custom scenarios (32) are the community-extensibility bet; a schema without a good guide is dead on arrival. The guide teaches by dissecting `intro-cell` (30) and excerpts of `locked-lab` (31).

## Scope

- Guide structure: (1) 10-minute "your first room" tutorial building a 2-room scenario from scratch, each step ending with `mcparcade validate-scenario`; (2) schema reference written from §14.2 + issue 28's normative payload table (every field, type, cap, with one-line examples — including the items-are-portable-entities rule); (3) design patterns: dependency chains, red herrings, fair hint tiers, par calibration method (the optimal-transcript counting rule from 30/31), lose-endings that feel earned; (4) anti-patterns: softlocks (validator catches unreachable wins, but conditional softlocks are the author's job — checklist provided), ambiguity in entity names, prose that addresses the AI (see T8 section), oversized text; (5) testing your scenario with an agent + reading the replay; (6) sharing & trust: the §14.8 trust note verbatim (installing a scenario means trusting its author; validation caps sizes but does not referee prose), license note (content you publish is yours; suggest CC-BY), installation instructions for players.
- A downloadable starter template `docs/guides/templates/starter-scenario.json` (valid, minimal, annotated via the `_comment` string fields that issue 28's schema explicitly tolerates and ignores).

## Detailed Requirements

1. Every JSON example in the guide must validate (a doc-test script `tools/check-guide-examples.ts` extracts fenced `json` blocks marked `<!-- validate -->` and runs the validator — wired into `npm test`).
2. The tutorial's final scenario is committed as a fixture and playable via harness (guide never drifts from reality).
3. Reference tables list the exact caps (§14.3) and every condition/effect kind with semantics (§14.4 turn-accounting rules included — authors must know `examine` is free).
4. T8 section is explicit and honest about enforcement: content is fiction addressed to the player character; meta-instructions to assistants must not be written (this is an authoring norm — the validator does not referee prose, §14.8); the repo's own bundled content is additionally checked by the heuristic tripwire in issue 39, which the guide mentions as an example check authors can copy.
5. Length ≤ 1 500 lines; navigable TOC.

## Acceptance Criteria

- [ ] Doc-test script green: all marked examples validate; tutorial scenario plays to win via a committed transcript test.
- [ ] Starter template validates (including its `_comment` fields) and is referenced from README's custom-content line.
- [ ] Fresh-reader test with concrete evidence: a non-author agent (via `codex exec`) or human follows the tutorial alone and produces a scenario committed to `test/fixtures/guide-follow/scenario.json` that passes `validate-scenario` with zero errors and plays to a win via the harness; friction notes recorded in the PR (pass = zero errors + winnable; any step where the reader had to consult files outside the guide is logged as friction to fix).

## Validation

- `npm test` includes the guide example checks; tutorial-follow session recorded in PR.

## Dependencies

- 28, 29, 30 (dissection target), 31 (excerpts).

## Non-goals

- Authoring GUI/tooling, scenario registry/sharing platform (v2), translations.

## Design References

- DESIGN.md §14 (all, esp. §14.8 trust note), §18.2 (T7, T8)
