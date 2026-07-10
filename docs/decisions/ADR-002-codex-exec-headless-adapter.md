# ADR-002: Agent integration via `codex exec` headless adapter

- Status: Accepted (2026-07-08, confirmed with product owner)

## Context

Handsfree must drive coding agents. Integration candidates:

1. Headless CLI runs (`codex exec --json`, resumable threads).
2. Driving an interactive TUI session via PTY/tmux scraping.
3. Vendor SDKs / server APIs per agent.

The owner's prior agent-orchestration architecture already **abandoned tmux/TUI
control in favor of CLI execution** (fragile scraping, unowned state); that
decision is treated as precedent.

## Decision

Define an `AgentAdapter` protocol; v1 ships exactly one implementation,
`CodexExecAdapter`, which spawns `codex exec`/`codex exec resume` per turn with
`--json` (JSONL events), `--output-schema` (structured final response), and
per-turn sandbox flags. All agent-side actions — including `git push` and PR
creation — go through agent turns under the tier policy (ADR-004); the app
executes no repo-affecting commands itself in v1.

## Rationale

- JSONL events + resumable thread ids give exactly the observability and
  multi-turn continuity the dialogue loop needs, with a stable public contract
  (verified against codex-cli 0.141.0; see research 2026-07-08).
- PTY scraping is version-fragile and cannot express sandbox tiers.
- A single execution path (the adapter) keeps the approval state machine and
  the audit trail unified; app-executed git verbs would duplicate execution,
  error handling, and policy in a second path.

## Consequences

- Mid-turn interactive approval is impossible in exec mode; the design uses
  `blocked` → approve → escalated resume instead (DESIGN §6.3–6.5).
- Codex CLI version drift is a live risk: preflight pins tested versions, the
  decoder tolerates unknown event types, fixtures are recorded from real runs.
- Additional agents (Claude Code, etc.) are v2 adapter implementations behind
  the same protocol.
