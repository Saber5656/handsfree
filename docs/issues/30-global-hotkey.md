# Title

Global hotkey manager and shortcut recorder control

## Summary

Implement the standalone hotkey component in `HandsfreeApp`: the `KeyMap`
table, `HotkeyBinding` model + validator, a Carbon `RegisterEventHotKey`
manager (no Accessibility permission), and the SwiftUI recorder control —
all consumable by 29 (boot registration) and 32 (Settings integration).

## Context

R6 makes the hotkey the sole session trigger; DESIGN §9.3 commits to zero
Accessibility permission, ruling out CGEventTap/global NSEvent monitors.
Default ⌃⌥Space has a known collision risk (input sources — known unknown
#5). This issue is deliberately config-free: 32 wires it to the ConfigStore,
29 wires boot registration and error surfacing.

## Scope

- `KeyMap`, `HotkeyBinding`, `HotkeyValidator`, `HotkeyManager`,
  `HotkeyRecorderView`. Not: config integration (32), boot/menu surfacing
  (29).

## Detailed Requirements

1. Model + table:
   ```swift
   public struct HotkeyBinding: Codable, Equatable, Sendable {
       public let key: String            // canonical names: "A"…"Z", "0"…"9",
                                         // "F1"…"F12", "Space", "Return", "Escape",
                                         // "Up","Down","Left","Right"
       public let modifiers: [String]    // canonical order: control, option, command, shift
   }
   public enum KeyMap {
       static func carbonKeyCode(for key: String) -> UInt32?
       static func carbonModifiers(for mods: [String]) -> UInt32?
       static func displayGlyphs(_ b: HotkeyBinding) -> String   // "⌃⌥Space"
   }
   public enum HotkeyValidationError: Error, Equatable {
       case unknownKey(String), unknownModifier(String)
       case noModifier                    // ≥1 of ⌃⌥⌘ required (⇧ alone insufficient)
       case duplicateModifiers
   }
   public enum HotkeyValidator { static func validate(_ b: HotkeyBinding)
       -> HotkeyValidationError? }        // also canonicalizes order/case
   ```
   Default binding constant `HotkeyBinding(key: "Space",
   modifiers: ["control","option"])` exported for 03's defaults; a doc note
   records the ⌃⌥⌘Space fallback recommendation for input-source-switching
   users (known unknown #5).
2. `HotkeyManager`:
   ```swift
   public protocol HotkeyRegistrar {      // seam: production = Carbon
       func register(keyCode: UInt32, modifiers: UInt32,
                     handler: @escaping () -> Void) throws -> HotkeyToken
       func unregister(_ t: HotkeyToken)
   }
   public final class HotkeyManager {
       public init(registrar: any HotkeyRegistrar = CarbonHotkeyRegistrar())
       public func apply(_ binding: HotkeyBinding) throws   // validates, unregisters old, registers new
       public func unregister()
       public var pressed: AsyncStream<Void>
       public private(set) var registrationState: RegistrationState
           // .unregistered | .registered(HotkeyBinding) | .failed(HotkeyBinding, osStatus: Int32)
   }
   ```
   Carbon path: `RegisterEventHotKey` + `InstallEventHandler`; failure maps
   to `.failed` with the OSStatus (macOS does not report the conflicting
   owner — callers phrase it as "likely in use; choose another").
3. `HotkeyRecorderView` (SwiftUI, self-contained): click-to-arm; captures
   the next keyDown via a LOCAL `NSEvent.addLocalMonitorForEvents` (no
   permissions); Esc cancels arming; renders `displayGlyphs`; invalid
   captures show the `HotkeyValidationError` message inline; emits the valid
   binding via a callback (`onCapture: (HotkeyBinding) -> Void`) — it does
   NOT write config itself.
4. Automated tests use a `FakeHotkeyRegistrar` (records register/unregister,
   injectable failure, synthesizable presses): apply/re-apply/unregister
   lifecycle, `pressed` delivery, failure state, validator table
   (round-trip every key/modifier; invalid names; modifier canonicalization;
   no-modifier and Space-without-modifier rejection).
5. Real-Carbon verification is a manual item executed in issue 29's launch
   checklist (frontmost-app matrix + no-Accessibility proof) — noted here,
   owned there.

## Acceptance Criteria

- [ ] KeyMap round-trip tests for every table entry; glyph rendering test.
- [ ] Validator: all four error cases + canonicalization tested.
- [ ] Manager lifecycle with the fake registrar: apply→re-apply unregisters
      the old token; failure → `.failed` with status; `pressed` fires on
      synthesized events.
- [ ] Recorder: arming, Esc cancel, invalid-combo inline error, callback
      payload (ViewInspector-free logic tests via an extracted
      `RecorderModel`).
- [ ] No Accessibility-requiring API anywhere in the component (grep for
      `CGEventTap`/`addGlobalMonitor` → nothing).

## Validation

`swift test --filter 'KeyMapTests|HotkeyValidatorTests|HotkeyManagerTests|RecorderModelTests'`.

## Dependencies

01.

## Non-goals

Hold-to-talk, multiple bindings, media keys, wake word, config persistence
(32), boot wiring (29).

## Design References

DESIGN.md §8.1, §8.3, §9.3, §16 (#5); R6; ADR-010 (no helper libs).
