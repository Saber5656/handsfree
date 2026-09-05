# Title

App lifecycle: launch at login, single instance, quit-with-tasks drain

## Summary

Implement the remaining lifecycle behaviors: `SMAppService` launch-at-login
(activating issue 32's toggle), single-instance enforcement via a local
alert, the quit flow protecting running tasks (cancel vs drain), and
deterministic teardown ordering.

## Context

A resident voice app must come back after reboot when asked, never run
doubled, and never kill a half-finished agent turn without consent (DESIGN
§8.1, §12). Drain semantics follow DESIGN §6.6: only `running` tasks block
quit — awaiting/pending states persist and resume after relaunch.

## Scope

- Login item service + reconcile, instance guard, quit flow + drain mode,
  teardown ordering. Recovery-on-start is 26 (called from 29's boot; this
  issue verifies the ordering).

## Detailed Requirements

1. **Launch at login**: bind `general.launch_at_login` to
   `SMAppService.mainApp` behind a seam (`LoginItemServicing`:
   register/unregister/status). Reconcile at boot and on config changes:
   config on + `.notRegistered` → re-register; user disabled it in System
   Settings (`.notFound`/`.notRegistered` while config on after a register
   attempt) → set config false + log. Issue 32's toggle now takes real
   effect; a status-mismatch caption in that tab renders from the service
   state (small, listed VM change included here).
2. **Single instance**: at launch, if another `NSRunningApplication` with
   `AppIdentity.bundleID` (excluding self) exists → show a plain `NSAlert`
   ("Handsfree is already running.") and terminate self with exit 0 after
   dismissal. No notification dependency; tolerate the app-translocation
   case (compare by bundle id only).
3. **Quit flow** (`applicationShouldTerminate` — menu Quit, logout,
   shutdown):
   - No `running` tasks → end active session, teardown, `.terminateNow`.
   - `running` tasks exist → app-modal NSAlert "N task(s) are still
     running": `Finish Tasks Then Quit` (default) / `Cancel Tasks and Quit`
     (destructive) / `Don't Quit`.
   - **Drain mode** (`Finish…`): reply `.terminateLater`; refuse new
     sessions (hotkey → spoken `error.quitting_drain` + earcon); menu shows
     "Quitting after N task(s)… (Cancel Quit)"; menu icon gains a badge.
     Drain completes when NO task remains in `running` state — tasks
     reaching `awaiting_input`/`awaiting_approval` do NOT block quit (they
     persist and resume after relaunch; DESIGN §6.6). Then
     `replyToApplicationShouldTerminate(true)`. `Cancel Quit` aborts drain
     (`…(false)`), restoring normal operation.
   - `Cancel Tasks and Quit`: `TaskManager.cancel` all running (SIGINT chain
     14/18), wait ≤ 8 s for terminal outcomes (injected clock), then
     terminate regardless.
   - Logout/shutdown takes the same alert path; if the system force-kills
     anyway, issue 26's recovery marks tasks `interrupted` on next launch
     (chain covered by the manual scenario below).
4. **Teardown ordering** (single `Teardown.run()` used by every exit path):
   end session (FSM → ending) → arbiter stop → audio engine stop/release →
   transcript session flush/close → task index flush. An integration test
   with spy subsystems asserts the exact order.
5. Sleep/wake: no special handling (children survive sleep, DESIGN §12);
   log a line on `NSWorkspace.didWakeNotification` for diagnosability.

## Acceptance Criteria

- [ ] Login item: seam tests for register/unregister/reconcile paths (all
      four combinations); manual: toggle → appears in System Settings >
      Login Items; survives reboot (manual note).
- [ ] Second launch: `open -n` twice → alert shown, second instance exits 0,
      first instance unaffected (manual + automated instance-detection unit
      test with a seam).
- [ ] Quit matrix (integration tests with FakeCodex): no-tasks immediate;
      drain completes when `slow-drip` ends; drain does NOT wait for a task
      that lands in `awaiting_input` (FakeCodex `needs-input`); cancel-quit
      restores sessions; Cancel-Tasks path terminates within 8 s.
- [ ] Drain refuses new sessions with the spoken notice (spy assertion).
- [ ] Teardown-order spy test green.
- [ ] Manual: `kill -9` during a running task → relaunch marks it
      `interrupted` and pending (documented run with 26's recovery).

## Validation

`swift test --filter 'LifecycleTests|QuitFlowTests'`; manual checklist
(login item, double-launch, kill -9, logout) in PR.

## Dependencies

26, 29, 32.

## Non-goals

Auto-relaunch after crash, Sparkle/updates, fast-user-switching hardening
(log-only), notification-based second-instance notice.

## Design References

DESIGN.md §8.1, §3.2, §6.6, §12; ADR-009; issue 19 (`error.quitting_drain`).
