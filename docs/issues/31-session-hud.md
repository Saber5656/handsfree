# Title

Session HUD: floating non-activating panel with approval controls

## Summary

Implement the session HUD (DESIGN §8.2): a floating `NSPanel` mirroring the
orchestrator state snapshot — live transcription, narration lines, task
digests, the demo-mode badge, and the Tier-3 Approve/Deny buttons — without
ever taking keyboard focus.

## Context

The HUD is the visual complement of the voice loop and part of the security
UI: the Tier-3 screen confirm is a **pointer-only** interaction by design
(amended DESIGN §5.4 — the panel never becomes key, which is itself the
anti-spoofing property). Action text arrives pre-sanitized (≤ 120 chars)
from the orchestrator's boundary (28).

## Scope

- Panel window controller + SwiftUI content + `HUDViewModel` + routing of
  approval clicks and menu `acknowledge` actions. Not: approval logic (21),
  notification routing (35).

## Detailed Requirements

1. Window: `NSPanel` with `.nonactivatingPanel`, `.utilityWindow`,
   `level = .floating`, `collectionBehavior = [.canJoinAllSpaces,
   .fullScreenAuxiliary]`, `becomesKeyOnlyIfNeeded = true` (and content
   avoids first responders entirely), no title bar,
   `NSVisualEffectView` material background, width 360 pt, height auto
   ≤ 240 pt. Draggable by background; `setFrameAutosaveName("HandsfreeHUD")`;
   default top-right 16 pt margins; if the saved frame is offscreen at show
   time, reposition onto the main screen (tested manually on a
   display-config change).
2. Visibility: appears on session start; hides 5 s after session end; also
   shows (read-only) when issue 29's menu `acknowledge(taskID)` action fires,
   displaying that task's digest for 8 s. Never shown at app launch without
   a session.
3. Content (renders `HUDViewModel`):
   - state chip (localized state name + mic glyph consistent with 29's
     table) + **"Demo" badge when `mode == .demo`** (34's requirement);
   - bound task + project line ("#2 · handsfree");
   - live area: current volatile transcription (dimmed) OR last final
     utterance;
   - narration list: last 3 `HUDLine`s (23);
   - result digest after completion: voice_summary text; `detail` opens a
     scrollable read-only popover — control characters stripped, no
     markdown rendering (v1);
   - error banner for `error_recoverable`.
4. Approval section (drives DESIGN §5.1 approval stages):
   - shield icon, the sanitized action text (already ≤ 120 — the view model
     additionally asserts length and strips control chars, defense in
     depth), tier label ("Network" orange / "Full access" red);
   - stage rendering: `announce`/`awaitEcho` → expected-phrase caption
     (「承認 4 9」) with **no enabled buttons**; `awaitScreenConfirm`
     (Tier 3 only) → `Approve` / `Deny` buttons (min 44 pt), clicks call
     `orchestrator.screenConfirm(true/false)` (28's API);
   - Tier 2 never shows buttons (voice-only); denial/timeout states clear
     the section.
5. All strings via `L10n` (29's parity test extends automatically).
6. `HUDViewModel.make(snapshot: SessionStateSnapshot) -> HUDViewModel` —
   pure builder unit-tested for every FSM state, both approval tiers, all
   three approval stages, demo mode, and the acknowledge-display mode.

## Acceptance Criteria

- [ ] Model-builder tests: every FSM state; approval matrix (T2: no buttons
      in any stage; T3: buttons enabled ONLY in awaitScreenConfirm; denied/
      timeout clears); demo badge; acknowledge display.
- [ ] Injection test: hostile action text (control chars/overlong) rendered
      inert and truncated by the view-model assertion layer.
- [ ] Manual: HUD never activates the app (type in TextEdit while the HUD
      updates; caret stays in TextEdit) — screen recording in PR.
- [ ] Manual approval flow with FakeCodex `blocked-full`: action text
      shown; Approve click completes the escalated resume; Deny denies.
      With `blocked-network` (T2): section shows phrase caption, no buttons.
- [ ] Frame persists across relaunch; offscreen-recovery works (manual).
- [ ] ja + en render without truncation at 360 pt (screenshots).

## Validation

`swift test --filter HUDViewModelTests`; manual checklist + recordings in
the PR.

## Dependencies

28, 29.

## Non-goals

Transcript history browser, markdown rendering, keyboard operation of
approval buttons (pointer-only by amended design), notification-driven
display (35 routes through the orchestrator API instead).

## Design References

DESIGN.md §8.2, §5.1, §5.4 (amended), §9.2 (T2/T3); ADR-004; issue 28
(state snapshot API).
