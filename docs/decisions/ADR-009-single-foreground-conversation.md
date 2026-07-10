# ADR-009: Single foreground conversation, numbered background tasks

- Status: Accepted (2026-07-08, confirmed with product owner)

## Context

The owner's daily workflow runs multiple coding agents in parallel. A full
voice multiplexer (switch/query tasks by name at any time) explodes the FSM,
phrase grammar, and notification arbitration. R8 fixed the v1 boundary.

## Decision

- Exactly one voice conversation at a time, bound to at most one task.
- Dispatched turns keep running when the session ends (they belong to the
  TaskManager, not the session). Terminal/blocked transitions announce via
  notification + speech.
- Re-engagement is number-based and prompt-driven: at session start, pending
  tasks (awaiting input/approval/unacknowledged results) are enumerated and
  bound via `yes` / 「タスク2」-style selection (DESIGN §6.6). Free-form
  name-based switching mid-session is v2.

## Rationale

Keeps the v1 state machine tractable and testable while preserving the two
essential parallel behaviors: long tasks don't hold the microphone hostage,
and an agent's question never dead-ends hands-free re-entry.

## Consequences

- `max_concurrent_tasks` (default 3) guards runaway parallelism.
- Task ids must be short daily-monotonic integers so they are speakable.
- v2 named switching will extend `task_select` rather than reshaping the FSM.
