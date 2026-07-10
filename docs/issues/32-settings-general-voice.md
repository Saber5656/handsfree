# Title

Settings window scaffold + General and Voice tabs

## Summary

Build the Settings window frame (tabbed, SwiftUI) and its first two tabs:
General (hotkey, language, idle timeout, launch-at-login toggle) and Voice
(STT locale assets, TTS voice pickers with quality labels + preview, rate,
verbosity).

## Context

DESIGN §8.3 fixes the tab set. Settings binds directly to the ConfigStore
(03) — every control is a thin view over config keys, applied live (no
Save button). The Voice tab is also the recovery surface for the
compact-voice / missing-asset situations found in research.

## Scope

- Window scaffold + General + Voice tabs. Projects/Policy/Privacy/Agent tabs
  are issue 33. Diagnostic snapshot button included here (About area).

## Detailed Requirements

1. Scaffold: standard Settings scene (⌘, from menu bar), tabs: General,
   Voice, Projects, Policy, Privacy, Agent, About — tabs 3–6 render
   "placeholder, see issue 33" stubs in this PR. Window 560×420 pt, fixed.
2. Config binding layer: a small `ConfigBinding<Value>` helper bridging
   SwiftUI `Binding` ⇄ `ConfigStore.update` (async write-behind, immediate UI
   echo, error toast on write failure) — reused by 33; unit-tested with the
   store from 03.
3. General tab:
   - Hotkey: `HotkeyRecorderView` (30) + current-binding glyph + reset-to-
     default; registration errors surface inline.
   - Language mode: picker `auto / 日本語 / English`
     (`general.locale_mode`) with caption explaining it affects speech, not UI.
   - Idle timeout: stepper 10–300 s.
   - Launch at login: toggle bound to `general.launch_at_login` (the
     `SMAppService` side-effect is issue 36; until merged the toggle persists
     config only — note in code).
4. Voice tab:
   - STT section per language (ja/en): status line from
     `STTProvider.availability` (`Installed ✓ / Download required (size) /
     Unauthorized / Unsupported`), Download button driving `prepare()` with
     determinate progress (08 `preparationProgress`), error presentation
     (no-network case).
   - TTS section per language: voice picker listing `voices(for:)` grouped by
     quality with labels `Compact/Enhanced/Premium`; a caption when only
     Compact voices exist: "Better voices can be downloaded in System
     Settings → Accessibility → Spoken Content" + "Open System Settings"
     button (`x-apple.systempreferences:com.apple.preference.universalaccess`
     best-effort URL; fallback: open the Settings app root and show
     instructions — implement the fallback, macOS URL anchors are brittle).
   - Preview button per language: speaks a fixed localized sample through the
     production arbiter (respects half-duplex, priority `.result`).
   - Rate slider 0.1–1.0 (live preview on release); verbosity picker
     quiet/milestones/verbose with one-line descriptions.
5. About area (within scaffold): app version (VERSION resource), macOS
   version, "Copy diagnostic snapshot" button (04's builder → pasteboard).
6. All strings via `L10n` (en/ja).

## Acceptance Criteria

- [ ] `ConfigBinding` round-trip test (UI change → config file change →
      external change → UI update via `changes` stream).
- [ ] Voice tab reflects real availability states (manual matrix: ja
      installed / en not-yet — screenshots) and download progress works
      (`live` manual).
- [ ] Missing-voice degradation note appears when configured voice id is
      absent (10's `lastResolutionNote` surfaced).
- [ ] Preview audible in ja and en with selected voices (manual).
- [ ] Diagnostic snapshot lands on the pasteboard, redacted (paste sample in
      PR with a planted fake token proven masked).
- [ ] en/ja parity test still green.

## Validation

`swift test --filter ConfigBindingTests SettingsModelTests`; manual checklist
with screenshots (both locales).

## Dependencies

03, 08, 10, 30 (+ 04 for snapshot, 11 for preview path).

## Non-goals

Projects/Policy/Privacy/Agent tab content (33), theming, import/export of
settings.

## Design References

DESIGN.md §8.3, §7.2, §13; research doc (voice inventory, settings deep link
caveat).
