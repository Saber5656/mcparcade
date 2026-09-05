# Title

README overhaul and per-client quickstart guides

## Summary

Rewrite `README.md` and add `docs/guides/` per DESIGN.md §2: what mcparcade is (an arcade that turns your agent into an AI that plays games), 60-second quickstart, per-client setup (Claude Code, Claude Desktop, Codex CLI, generic MCP), spectator usage, records/ratings explanation, security posture summary, and SECURITY.md.

## Context

The README is the OSS front door and the claim the product is judged by. It must reflect ADR-001 (one server — replacing the current "collection of servers" line) and the §18 posture honestly. All English.

## Scope

- `README.md`: hero paragraph + demo GIF placeholder (media deferred; framed shot list committed as `docs/guides/media-plan.md`), feature bullets (6 games, records, replays, spectator, offline/secretless), quickstart, client config snippets, spectator screenshot placeholder, scoring/rating explainer table, security summary linking SECURITY.md, license/contrib links.
- `docs/guides/claude-code.md`, `claude-desktop.md`, `codex-cli.md`, `generic-mcp.md`: exact config JSON/commands, first-game walkthrough ("ask your agent to: play a game of shogi at easy difficulty"), troubleshooting (node version, data dir perms, spectator port).
- `docs/guides/self-hosting.md`: §17 summary + link to the runbook (38).
- `SECURITY.md`: supported versions, report channel (GitHub private vulnerability reporting), 90-day disclosure default, scope notes (local tool; spectator token model).
- `CONTRIBUTING.md` (brief): dev setup, test commands, adding a game (pointer to §6.1), adding a scenario (pointer to 42's guide).

## Detailed Requirements

1. Client config snippets — expected shapes below are the starting point; verify each against the client's current docs at implementation time and cite the doc URL in an HTML comment beside each snippet:
   - Claude Code: `claude mcp add mcparcade -- npx -y mcparcade` (CLI) and the `.mcp.json` equivalent `{"mcpServers": {"mcparcade": {"command": "npx", "args": ["-y", "mcparcade"]}}}`.
   - Claude Desktop: the same `mcpServers` JSON block in `claude_desktop_config.json` (document the macOS path `~/Library/Application Support/Claude/`).
   - Codex CLI: `~/.codex/config.toml` — `[mcp_servers.mcparcade]` with `command = "npx"`, `args = ["-y", "mcparcade"]`.
   - Generic MCP: the command line `npx -y mcparcade` plus a note that any stdio MCP client works; MCP Inspector invocation for smoke-testing.
2. Quickstart is honest about requirements: Node ≥ 20, macOS/Linux supported (Windows best-effort).
3. The "how ratings work" table states anchors are product constants, not human Elo (§10.2 wording).
4. Prompt-injection trust note for custom scenarios (§14.8) appears in both README (one line) and the self-hosting/custom-content sections.
5. No marketing claims that outrun the code: every feature bullet maps to a shipped issue; a pre-release checklist item cross-checks bullets against closed issues.
6. Tone: technical, concrete, first-person-plural avoided; examples over adjectives.

## Acceptance Criteria

- [ ] README render check (each item explicit): all headings appear in GitHub's auto-TOC, every table renders as a table, every code fence renders with its language, zero broken relative links (link-check pass), placeholders clearly marked.
- [ ] Each client guide walked through end-to-end on a real machine at least once — evidence in the PR = per-client checklist with a terminal transcript or screenshot each; Claude Code and Codex CLI are mandatory, Claude Desktop mandatory on macOS, generic via MCP Inspector; any unavailable client is explicitly marked "not verified" in both the PR and the guide itself.
- [ ] Quickstart verified against the not-yet-published package: `npm pack` then `npx ./mcparcade-<version>.tgz` from a clean directory behaves exactly as the README describes (registry-based `npx mcparcade` is re-verified in issue 43's release checklist).
- [ ] SECURITY.md present with reporting instructions; linked from README.
- [ ] Old "collection of mini-game MCP servers" phrasing replaced repo-wide (grep).
- [ ] All internal doc links resolve (link-check script or manual pass recorded).

## Validation

- Manual walkthrough checklist in the PR (per client); markdown lint (Biome or `markdownlint` dev-only) green.

## Dependencies

- 13, 21, 26, 31, 36, 38 (documents shipped behavior; draftable earlier, mergeable last).

## Non-goals

- Demo video/GIF production, website, blog post, Japanese translation (v2), CHANGELOG mechanics (43).

## Design References

- DESIGN.md §1, §2, §10, §14.8, §16–§18
