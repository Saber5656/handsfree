# Title

Global hotkey manager and shortcut recorder control

## Summary

Implement `HandsfreeApp/HotkeyManager` on Carbon `RegisterEventHotKey`
(no Accessibility permission) with a config-driven binding, plus the SwiftUI
recorder control used by Settings, and the key/modifier mapping table shared
with config validation (03).

## Context

R6 makes the hotkey the sole session trigger; DESIGN §9.3 commits to zero
Accessibility permission, which rules out CGEventTap/NSEvent global monitors
for activation. Default is ⌃⌥Space with a known collision risk (input-source
switching — known unknown #5), so rebinding must be first-class.

## Scope

- HotkeyManager, KeyMap table, RecorderView. Not: what the hotkey does (29/28
  route it), Settings tab assembly (32).

## Detailed Requirements

1. `KeyMap`: bidirectional table `String ⇄ (carbonKeyCode, displayGlyph)` for
   keys: letters A–Z, digits, F1–F12, Space, Return, Escape, arrows;
   modifiers `control/option/command/shift ⇄ carbon flags ⇄ ⌃⌥⌘⇧`. Exposed to
   HandsfreeCore config validation via a protocol-registered validator
   (03's hotkey validation upgrades from shape-only to table-backed in this
   PR — update 03's stub).
2. `HotkeyManager`:
   ```swift
   func register(_ binding: HotkeyBinding) throws   // unregisters previous
   func unregister()
   var pressed: AsyncStream<Void>
   ```
   - Carbon `RegisterEventHotKey` + `InstallEventHandler`; errors map to
     `HotkeyError.registrationFailed(osStatus)` — surfaced by Settings as
     "conflict likely, choose another" (best-effort; macOS doesn't report the
     owner).
   - Reacts to config changes (03 `changes` stream) — rebind live.
   - At least one modifier required (bare keys rejected in KeyMap validation;
     Space requires ≥ 1 of ⌃⌥⌘).
3. `HotkeyRecorderView` (SwiftUI): click-to-arm, captures the next local
   keyDown via `NSEvent.addLocalMonitor` (local only — no permissions),
   renders glyph string (⌃⌥Space), Esc cancels arming, invalid combos show
   the KeyMap error inline. Writes to config only on valid capture.
4. Default binding constant `⌃⌥Space` lives in the config defaults (03) and
   a doc note records the fallback recommendation (⌃⌥⌘Space) for JIS
   input-source users (known unknown #5).
5. Register at app boot AFTER config load; failure at boot → menu bar error
   badge + Settings hint, app still usable via menu (DESIGN §12 resilience).

## Acceptance Criteria

- [ ] KeyMap round-trip tests (string→code→string) for every entry; invalid
      strings rejected.
- [ ] Manual: default hotkey toggles a test log line with the assembled app in
      any frontmost app (TextEdit, Finder) — WITHOUT Accessibility permission
      granted (verify in System Settings that Handsfree is absent from the
      Accessibility list; screenshot in PR).
- [ ] Rebind via recorder updates config and takes effect without relaunch.
- [ ] Registration failure path (attempt binding ⌘Space while Spotlight owns
      it — or a second in-process registration) shows the typed error.
- [ ] `pressed` stream delivers on MainActor-independent context (test with a
      posted Carbon event if feasible; else covered by manual checklist).

## Validation

`swift test --filter KeyMapTests`; manual checklist (frontmost-app matrix,
no-accessibility proof) in the PR.

## Dependencies

01, 03 (validator upgrade touchpoint).

## Non-goals

Hold-to-talk mode, multiple bindings, media-key support, wake word.

## Design References

DESIGN.md §8.1, §8.3, §9.3, §16 (#5); R6; ADR-010 (no helper libs).
