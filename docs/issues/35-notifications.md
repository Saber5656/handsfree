# Title

Background task notifications

## Summary

Implement `HandsfreeApp/NotificationManager`: user notifications for unbound
task transitions with a "Start voice session" action that binds the task —
under the amended DESIGN §8.5 content policy (questions/actions are never
disclosed on the lock screen).

## Context

ADR-009's background continuation needs reliable re-engagement. TTS
announcements (26→28) cover the at-desk case; notifications cover the rest.
Content policy (T5, amended §8.5): completed tasks may carry the sanitized
`voice_summary`; needs-input / awaiting-approval notifications carry only
task number, project, and state. Routing uses the orchestrator public API
from issue 28.

## Scope

- NotificationManager + category/action wiring + routing + cleanup. Not:
  TaskEvent production (26), spoken announcements (28), HUD display (31).

## Detailed Requirements

1. Authorization: request `UNUserNotificationCenter` authorization
   (`.alert`, `.sound`) lazily at the FIRST background task event (not app
   launch); denied → one-time menu-bar hint via the boot-problem row
   mechanism (29), never re-prompt, System Settings link in the hint.
2. Category `HANDSFREE_TASK`, action `START_SESSION`
   ("Start voice session", `.foreground`), registered at launch.
3. Identifier scheme (cleanup depends on it): request identifier
   `handsfree.task.<taskID>.<monotonic-seq>`; `threadIdentifier =
   "handsfree.task.<taskID>"` (coalesces per task). The manager keeps a
   per-task list of delivered request identifiers.
4. Mapping from `TaskEvent` digests (26 — digests already exclude
   question/action/prompt text by construction; the manager asserts this
   type-level). Exact L10n keys + en reference strings:
   | Event | Title key (en reference) | Body |
   |---|---|---|
   | `.terminal(succeeded)` | `notif.done.title` ("Task #{id} complete — {project}") | sanitized voiceSummary ≤ 120, else `notif.done.body_generic` ("Finished.") |
   | `.terminal(failed)` | `notif.failed.title` ("Task #{id} failed — {project}") | `notif.failed.body` ("Open a session to hear what happened.") |
   | `.terminal(cancelled)` | `notif.cancelled.title` | `notif.cancelled.body` |
   | `.terminal(denied)` | `notif.denied.title` ("Task #{id} — approval declined") | `notif.denied.body` ("The task is paused and can be resumed.") |
   | `.becamePending(awaiting_input)` | `notif.question.title` ("Task #{id} has a question — {project}") | `notif.question.body` ("Start a voice session to hear and answer it.") |
   | `.becamePending(awaiting_approval)` | `notif.approval.title` ("Task #{id} awaits approval — {project}") | `notif.approval.body` ("Start a voice session to review and approve.") |
   | `.interrupted` | `notif.interrupted.title` | `notif.interrupted.body` ("It can be resumed from where it left off.") |
   No other fields ever appear (no `detail`, no nonce content — nonces are
   generated only in-session by 21 and cannot exist at notification time; a
   test documents this invariant).
5. Routing: `START_SESSION` and default click →
   `orchestrator.startSession(binding: .task(id))` (28's API). If a session
   is already active → play a brief earcon + HUD hint via the orchestrator
   (single-session invariant, ADR-009) — no second session.
6. Cleanup: on task acknowledge/bind, remove that task's delivered AND
   pending requests by stored identifiers
   (`removeDeliveredNotifications(withIdentifiers:)` +
   `removePendingNotificationRequests`).
7. Foreground presentation: delegate returns `.banner` so events surface
   even while other Handsfree UI is frontmost.
8. Seam: `UserNotificationCentering` protocol wrapping the center
   (authorization, add, remove, delegate callbacks) with a TestSupport fake
   — all mapping/routing/cleanup logic is unit-tested against it.

## Acceptance Criteria

- [ ] Mapping table: one test per row asserting title/body keys, thread id,
      request-id scheme, and the absence of question/action/detail content
      (planted-secret digest test).
- [ ] Lazy authorization: first event triggers exactly one request; denial
      path sets the one-time hint and suppresses future prompts.
- [ ] Routing: START_SESSION binds the right task via a spy orchestrator;
      active-session case takes the no-second-session path.
- [ ] Cleanup removes exactly the acknowledged task's identifiers.
- [ ] Manual: end a session with FakeCodex `slow-drip` running → completion
      banner arrives; click → session opens bound to the task (recording in
      PR).

## Validation

`swift test --filter NotificationManagerTests`; manual checklist in PR.

## Dependencies

20, 26, 28.

## Non-goals

Notification preference UI (System Settings suffices), custom sounds,
critical alerts, second-instance notice (36 uses an alert instead).

## Design References

DESIGN.md §8.5 (amended content policy), §6.6, §9.2 (T5); ADR-009; issue 28
(public API), 26 (digest guarantees).
