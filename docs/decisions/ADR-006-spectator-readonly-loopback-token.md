# ADR-006: Spectator UI is read-only, loopback-only, token-gated, SSE-based

Status: accepted (web UI in v1 = user decision, 2026-07-10; posture = this ADR)

## Context

v1 includes a local web UI for humans to watch games live and review replays. Any local HTTP server in a dev tool is a real attack surface: DNS rebinding, XSS from game text, other local users, token leakage. Werewolf adds hidden-information integrity (a human peeking at roles could coach the agent — and the NPC suspicion model's integrity depends on secrecy).

## Decision

- **Read-only**: GET-only JSON API + SSE; no action endpoints exist in the spectator (playing happens only through MCP).
- **Loopback + token**: bind 127.0.0.1; per-process random 128-bit token required on every request; Host/Origin allowlist; CSP `default-src 'self'`; zero external assets; no CORS (DESIGN.md §11.2–§11.3).
- **SSE, not WebSocket**: one-way live updates suit spectating; EventSource reconnects natively; no extra dependency (Node `http` suffices).
- **Public-view rule**: active hidden-info sessions expose public information only; full reveal after the game ends (§15.8).
- Cross-process visibility via filesystem watching, not a shared daemon (§11.5).

## Consequences

- The UI cannot be a cheating or remote-control channel even if the token leaks locally; worst case is read access to public game views on the same machine.
- SSE + refetch keeps the SPA dumb and the API surface tiny.
- Cost: no in-browser play (v1 non-goal), humans-as-players deferred to v2 lobbies (ADR-002 seam).

## Alternatives rejected

- WebSocket + write actions ("click to play as a human"): doubles the surface (auth for writes, CSRF, turn arbitration) for a non-goal.
- No auth ("it's only localhost"): multi-user machines and rebinding tricks make bare loopback HTTP insufficient; a token is nearly free.
