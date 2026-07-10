# Title

App lifecycle: launch at login, single instance, quit-with-tasks drain

## Summary

Implement the remaining application lifecycle behaviors: `SMAppService` launch
at login, single-instance enforcement, the quit flow that protects running
tasks (cancel vs drain), and clean teardown ordering.

## Context

A resident voice app must behave like good furniture: present after reboot
when asked (config toggle exists since 32), never doubled, and never killing a
half-finished agent turn without consent (DESIGN §8.1 quit warning, §12
failure catalog).

## Scope

- Login item service, instance guard, quit flow + drain mode, teardown.
  Recovery-on-start is 26; this issue calls it at the right moment (already
  wired in 29's boot — verify ordering here).

## Detailed Requirements

1. **Launch at login**: bind `general.launch_at_login` to
   `SMAppService.mainApp` register/unregister; reconcile at boot (config says
   on but service reports `.notRegistered` → re-register; user disabled it in
   System Settings → update config to false and log). Settings toggle (32)
   now takes real effect; status mismatch renders as a caption in that tab
   (small PR to 32's view model included here).
2. **Single instance**: at launch, if another `NSRunningApplication` with the
   same bundle id (excluding self) is running → activate nothing (LSUIElement),
   post a user notification "Handsfree is already running", terminate self
   with exit 0. Must tolerate the translocation/read-only case gracefully.
3. **Quit flow** (menu Quit, ⌘Q on any app window, logout/shutdown
   `applicationShouldTerminate`):
   - No running tasks → terminate immediately (end active session first:
     orchestrator teardown → audio engine released).
   - Running tasks (state `running`) → alert (NSAlert, app-modal):
     "N tasks are still running" with buttons
     `Finish Tasks Then Quit` (default) / `Cancel Tasks and Quit`
     (destructive) / `Don't Quit` (cancel).
   - **Drain mode** (`Finish…`): reply `.terminateLater`; refuse new sessions
     (hotkey → spoken "quitting after current tasks" + earcon); menu bar icon
     gains a badge and menu shows "Quitting after N task(s)… (Cancel Quit)";
     when the last running task reaches a terminal/pending-input state →
     complete termination (pending-input tasks do NOT block quit — they are
     safely resumable via thread id after relaunch; document).
     `Cancel Quit` menu item aborts drain (`replyToApplicationShouldTerminate(false)`).
   - `Cancel Tasks and Quit`: `TaskManager.cancel` all running (SIGINT chain
     from 14/18), wait ≤ 8 s for terminal outcomes, then terminate.
   - Logout/shutdown: same alert; system may force-quit on timeout — recovery
     (26) handles the `interrupted` marking on next launch (test this chain).
4. **Teardown ordering** (single place, called from all exits):
   end session (FSM → ending) → arbiter stop → audio engine stop/release →
   transcript session records flushed → task index flushed. Assert order via
   an integration test with spy subsystems.
5. Sleep/wake: no special handling beyond DESIGN §12 (children survive sleep);
   add a log line on wake for diagnosability (`NSWorkspace.didWakeNotification`).

## Acceptance Criteria

- [ ] Login item: toggle on → present in System Settings > Login Items;
      reboot-persistence manual check; reconcile paths unit-tested with an
      SMAppService seam.
- [ ] Second launch exits cleanly with the already-running notification
      (manual: `open -n` twice).
- [ ] Quit alert matrix: no-tasks, running+finish (drain completes on
      FakeCodex `slow-drip` end), running+cancel (≤ 8 s), don't-quit —
      integration tests with FakeCodex + manual pass.
- [ ] Drain mode refuses new sessions with the spoken notice; Cancel Quit
      restores normal operation (test).
- [ ] Kill -9 during a running task → next launch marks it `interrupted` and
      pending (chain test with 26).
- [ ] Teardown-order spy test green.

## Validation

`swift test --filter LifecycleTests QuitFlowTests`; manual checklist
(login item, double-launch, logout path) in PR.

## Dependencies

26, 29 (+ small 32 view-model amendment).

## Non-goals

Auto-relaunch after crash, Sparkle/updates, multi-user/fast-user-switching
hardening (log-only).

## Design References

DESIGN.md §8.1, §3.2, §12; ADR-009.
