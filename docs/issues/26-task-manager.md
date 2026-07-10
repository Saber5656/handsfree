# Title

TaskManager: task lifecycle, background continuation, pending queue, recovery

## Summary

Implement `HandsfreeCore/Tasks/TaskManager`: the actor that owns all
`TaskRecord`s, enforces the single-binding rule and concurrency cap, keeps
tasks alive across session end, persists the index, announces background
transitions, and recovers after app restart.

## Context

ADR-009/DESIGN §6.6 define the concurrency model: one foreground conversation,
detached background turns, numbered re-engagement. This actor is the bridge
between the session FSM (foreground) and running `RunningTurn`s (background).

## Scope

- Task model + manager + persistence via 27's index + tests. Not: FSM wiring
  (28), notifications UI (35 — consumes this actor's event stream).

## Detailed Requirements

1. `TaskRecord` exactly per DESIGN §6.6 (states `running, awaiting_input,
   awaiting_approval, succeeded, failed, cancelled, denied, interrupted`)
   plus `acknowledged: Bool` (set when the user hears/binds the result) and
   `spawnPID/spawnStartedAt` for recovery.
2. IDs: small integers, monotonic per calendar day (1, 2, 3…), reset daily;
   speakable and shown in HUD/menu (「タスク2」). Collision-free via the
   persisted index.
3. API surface:
   ```swift
   func create(project: ProjectEntry, initialPrompt: String) async throws -> TaskID   // enforces cap
   func attachTurn(_ id: TaskID, turn: RunningTurn) async
   func turnEnded(_ id: TaskID, outcome: TurnOutcome) async      // state mapping + event
   func bind(_ id: TaskID) async throws     // single binding invariant
   func unbind() async
   var boundTask: TaskID? { get async }
   func pending() async -> [TaskDigest]      // awaiting_* or terminal-unacknowledged, oldest first
   func acknowledge(_ id: TaskID) async
   func cancel(_ id: TaskID) async           // forwards to turn.cancel()
   var events: AsyncStream<TaskEvent>        // .becamePending(digest), .terminal(digest), .interrupted(digest)
   ```
4. State mapping from `TurnOutcome`: contract `ok`→`succeeded`;
   `needs_input`→`awaiting_input` (store question); `blocked`→
   `awaiting_approval` (store blockedReason/action); `failed`→`failed`;
   `.cancelled`→`cancelled`; `.timedOut`→`failed(timeout)`. Denied approvals
   (from 21 via 28) → `denied` but REMAIN in `pending()` until acknowledged
   (user may resume with different instruction — DESIGN §5.4).
5. Cap: `agent.max_concurrent_tasks` (default 3) counts `running` only;
   `create` beyond cap throws `TaskError.tooManyTasks` (spoken guidance key).
6. Background events: transitions of UNBOUND tasks emit `TaskEvent`s; bound-
   task transitions do not (the FSM handles them inline) — dedup rule tested.
7. Persistence: every mutation writes the task index via TranscriptStore (27)
   atomically. Recovery on startup: tasks recorded `running` → check PID
   liveness with start-time match (`kill(pid,0)` + `proc_pidinfo` start time):
   live orphan → `kill` process group (we never re-attach; DESIGN §3.2) and
   mark `interrupted`; dead → `interrupted`. Interrupted tasks appear in
   `pending()` with a resume hint (thread_id retained for `codex exec resume`).
8. All timing (daily reset) via injected clock/calendar.

## Acceptance Criteria

- [ ] Single-binding invariant: second `bind` without `unbind` throws; bind of
      nonexistent/terminal-acknowledged task throws.
- [ ] Cap enforcement + spoken-error key test.
- [ ] State mapping table fully tested (7 outcomes → states, incl. stored
      question/action payloads).
- [ ] Pending ordering + acknowledged filtering tests.
- [ ] Bound vs unbound event dedup test.
- [ ] Recovery: fixture index with (a) dead PID (b) live decoy process
      (spawned sleeper with matching PID reuse simulated via start-time
      mismatch) → both `interrupted`, decoy NOT killed on start-time mismatch.
- [ ] Daily ID reset across a simulated midnight.

## Validation

`swift test --filter TaskManagerTests` (includes a live-process recovery test
using a locally spawned `/bin/sleep`).

## Dependencies

03, 12, 27.

## Non-goals

Voice task switching by name (v2), scheduling/queueing beyond the cap,
retrying failed turns automatically (HOTL principle: the user decides).

## Design References

DESIGN.md §6.6, §3.2, §5.4, §12; ADR-009.
