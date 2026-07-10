# Title

Session HUD: floating non-activating panel with approval controls

## Summary

Implement the session HUD (DESIGN §8.2): a small floating `NSPanel` that
mirrors session state, live transcription, recent narration, and hosts the
Tier-3 Approve/Deny buttons — without ever stealing keyboard focus.

## Context

The HUD is the visual complement to a voice-first loop: it must inform
(especially during approvals: exact action text) while never interfering with
the user's active app (`.nonactivatingPanel`). The Tier-3 screen confirm
(ADR-004) lives here, so part of this surface is security UI.

## Scope

- Panel window controller + SwiftUI content + approval interaction plumbing
  to the orchestrator. Not: approval logic (21), menu bar (29).

## Detailed Requirements

1. Window: `NSPanel` with `.nonactivatingPanel`, `.utilityWindow`,
   `level = .floating`, `collectionBehavior = [.canJoinAllSpaces,
   .fullScreenAuxiliary]`, no title bar, rounded material background
   (`NSVisualEffectView`), fixed width 360 pt, height auto (≤ 240 pt).
   Draggable by background; frame persisted via
   `setFrameAutosaveName("HandsfreeHUD")`; default position top-right with
   16 pt margins.
2. Visibility: appears on session start, hides 5 s after session end;
   also shown while a background-task notification interaction is pending
   bind. Never appears at app launch without a session.
3. Content (SwiftUI, observes orchestrator state):
   - state chip (localized state name + mic glyph matching §8.1 states);
   - bound task + project line ("#2 · handsfree");
   - live area: current volatile transcription (dimmed, updates in place)
     OR last final utterance;
   - narration list: last 3 `HUDLine`s (23), icons per item type;
   - result digest after completion (voice_summary text; `detail` opens a
     scrollable read-only popover — control characters stripped, no markdown
     rendering in v1);
   - error banner row for `error_recoverable` states.
4. Approval mode (drives §5.1 `await_screen_confirm`):
   - prominent section: shield icon, **full sanitized action text**, tier
     label ("Full access" red / "Network" orange), and for Tier 3 two buttons
     `Approve` / `Deny` (min 44 pt height). Buttons are ENABLED only in
     `await_screen_confirm` stage; clicks emit `screenConfirm(true/false)`
     into the orchestrator.
   - During `await_echo` the section shows the expected phrase visually
     (「承認 4 9」) as a caption — aiding recall without weakening the voice
     check.
   - Panel must NOT accept keyboard focus even in approval mode (mouse only —
     `becomesKeyOnlyIfNeeded`, buttons via mouse; document the deliberate
     keyboard exclusion with a threat-model reference).
5. All strings via `L10n` (parity test from 29 extends automatically).
6. Testability: HUD renders a `HUDViewModel` built by a pure
   `HUDViewModel.make(state:, task:, lines:, approval:)` — unit-test the
   model builder for every FSM state incl. both approval stages.

## Acceptance Criteria

- [ ] Model-builder tests cover every session state (chip text, sections
      shown/hidden, button enablement matrix).
- [ ] Manual: HUD never activates the app (type in TextEdit while HUD
      updates; caret never leaves TextEdit) — checklist + screen recording.
- [ ] Approval flow manual test with FakeCodex `blocked-full`: action text
      shown verbatim (sanitized), Approve click completes the escalated
      resume; Deny denies.
- [ ] Frame persists across relaunch; multi-display safe (moves onto a
      visible screen if saved frame is offscreen — implement + manual check).
- [ ] ja + en render without truncation at 360 pt (screenshots).

## Validation

`swift test --filter HUDViewModelTests`; manual checklist + recordings in PR.

## Dependencies

28 (state/approval streams), 29 (L10n helper).

## Non-goals

Transcript history browser, resizable/rich markdown rendering, keyboard
operation of approval buttons (explicitly excluded — see Req 4).

## Design References

DESIGN.md §8.2, §5.1, §5.4, §9.2 (T2/T3); ADR-004.
