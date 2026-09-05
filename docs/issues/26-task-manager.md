# Title

TaskManager: task lifecycle, background continuation, pending queue, recovery

## Summary

Implement `HandsfreeCore/Tasks/TaskManager`: the actor owning all
`TaskRecord`s — single-binding rule, concurrency cap, background
continuation, sanitized digests for speech/notifications, persistence via the
task index, and PID+start-time crash recovery.

## Context

ADR-009 / DESIGN §6.6. Process identity comes from
`RunningTurn.processIdentity` (issues 12/14/18). Persistence uses issue 27's
atomic task-index primitives. Digests feed notifications (35) and speech, so
their content rules are security-relevant (T3/T5).

## Scope

- Task model + manager + recovery + tests. Not: FSM wiring (28),
  notification rendering (35).

## Detailed Requirements

1. `TaskRecord` (Codable; persisted via 27's `TaskIndexFile`):
   `{ id: Int, projectID: UUID, projectName: String, threadID: String?,
   state: TaskState, tierHistory: [RiskTier], createdAt/updatedAt: Date,
   lastVoiceSummary: String?, pendingQuestion: String?,
   pendingApproval: { action: String, targetTier: RiskTier }?,
   acknowledged: Bool, processIdentity: ProcessIdentity?,
   interruptedResumeHint: Bool }`.
   `TaskState = running | awaiting_input | awaiting_approval | succeeded |
   failed | cancelled | denied | interrupted`.
2. IDs: small integers, monotonic per calendar day (1, 2, 3…), daily reset
   via injected clock+calendar; collision-free via the persisted index.
3. API:
   ```swift
   func create(project: ProjectEntry, initialPrompt: String) async throws -> TaskID
   func attachTurn(_ id: TaskID, turn: RunningTurn) async     // captures processIdentity
   func turnEnded(_ id: TaskID, outcome: TurnOutcome) async
   func approvalDenied(_ id: TaskID, reason: DenialReason) async
   func bind(_ id: TaskID) async throws
   func unbind() async
   var boundTask: TaskID? { get async }
   func pending() async -> [TaskDigest]
   func acknowledge(_ id: TaskID) async
   func cancel(_ id: TaskID) async
   var events: AsyncStream<TaskEvent>
   ```
4. Outcome → state mapping (exact matrix, one test per row):
   | Input | New state | Stored payload |
   |---|---|---|
   | `.completed` + contract `ok` | `succeeded` | lastVoiceSummary |
   | `.completed` + contract `needs_input` | `awaiting_input` | pendingQuestion |
   | `.completed` + contract `blocked` | `awaiting_approval` | pendingApproval (action, target tier per §6.5 mapping) |
   | `.completed` + contract `failed` | `failed` | lastVoiceSummary |
   | `.completed`, contract nil (fallback) | `succeeded` | fallback summary, isFallback noted |
   | `.failed(reason)` | `failed` | reason key |
   | `.cancelled` | `cancelled` | — |
   | `.timedOut` | `failed` | timeout reason key |
   | `approvalDenied` | `denied` | pendingApproval retained (resumable) |
5. State table for `pending()` / `bind` / `acknowledge` / resume-eligibility
   (encode exactly):
   | State | in pending()? | bindable? | resumable thread? |
   |---|---|---|---|
   | running | no | yes (rebind to watch) | n/a |
   | awaiting_input | yes | yes | yes |
   | awaiting_approval | yes | yes | yes (approval flow) |
   | denied | yes until acknowledged | yes | yes (new instruction) |
   | succeeded/failed/cancelled | yes until acknowledged | yes (follow-up) | yes |
   | interrupted | yes until acknowledged | yes | yes (`interruptedResumeHint`) |
   `bind` throws on: nonexistent id, terminal+acknowledged, or another task
   already bound.
6. `TaskDigest` (the ONLY payload notifications/speech receive):
   `{ id, projectName, state, voiceSummary: String? (sanitized, ≤ 120),
   hasQuestion: Bool, hasApproval: Bool, updatedAt }` — NO question text,
   NO action text, NO prompt/detail/stderr (T5; 35's content policy).
   Sanitization via `SpeechTextSanitizer` happens when building digests.
7. Events: transitions of UNBOUND tasks emit
   `.becamePending(TaskDigest)` / `.terminal(TaskDigest)` /
   `.interrupted(TaskDigest)`; bound-task transitions emit nothing (FSM
   handles them inline) — dedup tested.
8. Cap: `create` beyond `max_concurrent_tasks` (counts `running` only) →
   `TaskError.tooManyTasks` (template `error.too_many_tasks`).
9. Persistence: every mutation writes via 27's atomic index API. Write
   failure policy: in-memory state stays authoritative; log once per
   session; retry on next mutation (tested with an injected failing index).
10. Recovery (startup): for each record with state `running`:
    - `processIdentity == nil` → `interrupted`.
    - `kill(pid, 0)` fails (ESRCH) → `interrupted`.
    - process alive: read current `proc_bsdinfo.pbi_start_tvsec/tvusec`;
      **exact match** with the stored values → orphan: `kill(-pgid, SIGKILL)`
      (never re-attach, DESIGN §3.2) then `interrupted`; mismatch (PID
      reuse) → do NOT kill; mark `interrupted`.
    All recovered records keep `threadID` and set `interruptedResumeHint`.

## Acceptance Criteria

- [ ] Outcome matrix: all 9 rows tested.
- [ ] State table: pending/bind/acknowledge behavior per state (incl. bind
      conflicts and terminal+acknowledged rejection).
- [ ] Digest contains no question/action/prompt text (type-level +
      content test with a planted secret in `detail` → absent).
- [ ] Bound vs unbound event dedup.
- [ ] Cap + template key.
- [ ] Recovery: (a) dead PID → interrupted; (b) live process with MATCHING
      start time (spawned `/bin/sleep` fixture registered in a fixture
      index) → killed + interrupted; (c) live process with MISMATCHED start
      time → NOT killed, interrupted. All three tested.
- [ ] Daily ID reset across simulated midnight; index write-failure policy
      test.

## Validation

`swift test --filter TaskManagerTests` (includes the live `/bin/sleep`
recovery tests — local processes, CI-safe).

## Dependencies

03, 12, 14, 27.

## Non-goals

Voice task switching by name (v2), scheduling beyond the cap, automatic
retry of failed turns (HOTL: the user decides).

## Design References

DESIGN.md §6.6, §3.2, §5.4, §6.5, §12, §9.2 (T5); ADR-009; issues 12/14
(ProcessIdentity), 27 (index primitives).
