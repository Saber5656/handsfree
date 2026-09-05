# Title

Settings window scaffold + General and Voice tabs

## Summary

Build the Settings scene (tabbed) and its first two tabs: General (hotkey
binding via issue 30's recorder, language, idle timeout, launch-at-login
toggle) and Voice (STT assets, TTS voice pickers + preview, rate,
verbosity) — all bound live to the ConfigStore through a reusable
`ConfigBinding` helper.

## Context

DESIGN §8.3 fixes the tab set. This issue owns the config↔UI binding layer
(reused by 33) and the hotkey config integration (30 is deliberately
config-free): it registers the semantic hotkey validator with 03's hook and
rebinds the `HotkeyManager` on config changes. The menu bar "Settings…" item
(29) opens this scene.

## Scope

- Settings scene + General + Voice tabs + `ConfigBinding` + hotkey config
  integration + diagnostic-snapshot button. Tabs Projects/Policy/Privacy/
  Agent are issue 33 (placeholder stubs here).

## Detailed Requirements

1. Scaffold: Settings scene (⌘,), tabs General / Voice / Projects / Policy /
   Privacy / Agent / About; tabs 3–6 render "see issue 33" stubs. Window
   560×420 pt fixed.
2. `ConfigBinding<Value>` (file
   `Sources/HandsfreeApp/Settings/ConfigBinding.swift`):
   ```swift
   @MainActor final class ConfigBinding<Value: Equatable>: ObservableObject {
       init(store: ConfigStore, get: @escaping (Config) -> Value,
            set: @escaping (inout Config, Value) -> Void)
       var binding: Binding<Value>       // immediate UI echo
   }
   ```
   Semantics (tested): writes are coalesced (200 ms debounce) into
   `store.update`; external changes from `store.changes` update the UI
   unless a local write is pending (local pending wins, then reconciles);
   store write failure → error toast via an injected `ErrorPresenting` seam
   and UI reverts to the store value.
3. Hotkey config integration (owns what 30 deferred):
   - register `HotkeyValidator` with 03's semantic-validation hook;
   - General tab embeds `HotkeyRecorderView`; `onCapture` writes the
     binding via ConfigBinding; a `ConfigStore.changes` observer calls
     `HotkeyManager.apply` live (no relaunch);
   - `registrationState == .failed` renders the inline "likely in use"
     error + reset-to-default button.
4. General tab: hotkey (above), language mode picker (auto/日本語/English —
   caption: affects speech, not UI), idle-timeout stepper 10–300 s,
   launch-at-login toggle bound to config (caption notes the system
   side-effect lands with issue 36; until then config-only).
5. Voice tab:
   - STT per language (ja/en): status from `availability` rendered through a
     seam (`STTStatusProviding`) so ALL FOUR states are testable
     (installed / download-required with size / unauthorized / unsupported);
     Download button drives `prepare()` with determinate progress from
     `preparationProgress` (08); no-network error presentation. Live
     screenshots are supplementary evidence, not the AC oracle.
   - TTS per language: voice picker grouped by quality (labels
     Compact/Enhanced/Premium); degradation note when the configured id is
     missing (10's `lastResolutionNote`); compact-only caption + "Open
     System Settings" button — implement as: try the
     `x-apple.systempreferences:com.apple.preference.universalaccess` URL,
     and on failure open System Settings plainly and show step-by-step text
     (the fallback IS the requirement; anchor URLs are brittle).
   - Preview buttons per language: speak a fixed localized sample through
     the production arbiter (priority `.result`, respects half-duplex).
   - Rate slider 0.1…1.0 (preview on release); verbosity picker with
     one-line descriptions.
6. About area: app version (VERSION resource), macOS version, "Copy
   diagnostic snapshot" — assembles `DiagnosticSnapshotInput` (04) from
   config JSON + task summary (26) + codex version (13 cache) and copies the
   redacted output to the pasteboard.

## Acceptance Criteria

- [ ] `ConfigBinding` tests: debounce, external-change precedence rules,
      write-failure revert + toast seam.
- [ ] Hotkey: rebind via recorder takes effect without relaunch (manual) and
      failure state renders inline (fake registrar test at the VM level).
- [ ] Voice tab VM tests cover all four STT availability states + download
      progress + missing-voice note.
- [ ] Preview audible in ja and en (manual); System-Settings fallback path
      shows instructions when the anchor URL fails (forced-failure test).
- [ ] Snapshot lands on the pasteboard redacted (planted fake token masked —
      paste sample in PR).
- [ ] General controls each persist to config.json (VM tests +
      one manual sweep); launch-at-login persists config only (36 note).
- [ ] en/ja parity test still green; screenshots of both tabs in both
      languages.

## Validation

`swift test --filter 'ConfigBindingTests|SettingsGeneralVMTests|SettingsVoiceVMTests'`;
manual checklist + screenshots.

## Dependencies

03, 04, 08, 10, 11, 30.
(Coordination note, not a prerequisite: the menu item that opens this scene
is rendered by issue 29.)

## Non-goals

Projects/Policy/Privacy/Agent tabs (33), `SMAppService` effect (36),
theming, settings import/export.

## Design References

DESIGN.md §8.3, §7.2, §13; research doc (voice inventory; settings deep-link
caveat); issues 04/08/10/30 APIs.
