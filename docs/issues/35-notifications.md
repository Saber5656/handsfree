# Title

Background task notifications

## Summary

Implement `HandsfreeApp/NotificationManager`: user notifications for unbound
task transitions (completed / needs input / awaiting approval / interrupted),
with a "Start voice session" action that binds the task, per DESIGN §8.5.

## Context

ADR-009's background continuation only works if the user reliably learns a
detached task needs them. TTS announcements (26→28) cover the at-desk case;
notifications cover away-from-audio and Do-Not-Disturb-later cases. Content
passes the same sanitizer as speech (threat T5: summaries only, never
`detail`).

## Scope

- NotificationManager + category/action wiring + routing into the
  orchestrator. Not: TaskManager events (26), spoken announcements (28).

## Detailed Requirements

1. Authorization: request `UNUserNotificationCenter` authorization
   (.alert, .sound) lazily — at the first background task creation, not app
   launch (less scary onboarding); denied → one-time menu bar hint, never
   re-prompt (link to System Settings).
2. Category `HANDSFREE_TASK` with actions:
   `START_SESSION` ("Start voice session", foreground action) and system
   dismiss. Registered at launch.
3. Mapping from `TaskEvent` (26) — one notification per event, thread
   id = task id (coalesces per task):
   - `.terminal(succeeded)`: title "Task #N complete — <project>", body =
     sanitized voice_summary (20; ≤ 120 chars);
   - `.terminal(failed/cancelled/denied)`: state-labeled title, body = short
     reason key;
   - `.becamePending(awaiting_input)`: title "Task #N has a question", body =
     sanitized question;
   - `.becamePending(awaiting_approval)`: title "Task #N awaits approval",
     body = sanitized blocked_action (NO nonce content — nonce is generated
     only when a session engages the approval; assert in tests);
   - `.interrupted`: title "Task #N was interrupted", body = resume hint.
4. Action routing: `START_SESSION` (and default click) →
   `orchestrator.startRequested(binding: .task(id))`; if a session is already
   active → play a brief earcon + HUD hint instead of a second session
   (single-session invariant, ADR-009).
5. Presentation while app frontmost: show banners (`.banner` presentation
   option) — the menu bar app is technically always "frontmost-less", but
   handle the delegate callback anyway.
6. Cleanup: delivered notifications for a task are removed when the task is
   acknowledged/bound (`removeDeliveredNotifications(withIdentifiers:)`).
7. Never include: `detail` content, file paths beyond basenames (sanitizer
   handles), nonce digits, project absolute paths.

## Acceptance Criteria

- [ ] Mapping table unit tests (event → title/body/thread id) for all five
      cases, incl. sanitization of a token-bearing summary (04/20 planted case).
- [ ] Nonce-absence assertion test for the awaiting_approval case.
- [ ] Routing test: START_SESSION binds the right task; active-session case
      takes the no-second-session path (orchestrator stub assertions).
- [ ] Acknowledge → delivered notification removed (manual + unit where
      mockable).
- [ ] Manual: end session with FakeCodex `slow-drip` running → completion
      banner arrives, click → session opens bound to the task (recording in PR).

## Validation

`swift test --filter NotificationMappingTests`; manual checklist in PR.

## Dependencies

20, 26 (+ orchestrator routing hook from 28).

## Non-goals

Notification preferences UI (system settings suffice in v1), sounds
customization, critical alerts.

## Design References

DESIGN.md §8.5, §6.6, §9.2 (T5); ADR-009.
