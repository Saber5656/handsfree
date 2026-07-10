# Title

App shell and menu bar controller

## Summary

Implement `HandsfreeApp`'s boot sequence and the `NSStatusItem` menu bar
presence: state-reflecting icon, the session/tasks menu, and localized UI
strings scaffolding (en/ja).

## Context

The menu bar is the only permanent UI (R1, DESIGN §8.1). It must truthfully
mirror the FSM state (threat T10: mic state visibility) and expose the
non-voice fallbacks (start/end session, task list, Settings).

## Scope

- `@main` app delegate, boot wiring, MenuBarController, Localizable.strings
  setup. Not: HUD (31), Settings content (32/33), hotkey internals (30).

## Detailed Requirements

1. Boot sequence (`AppDelegate.applicationDidFinishLaunching`):
   config load (03) → logging init (04) → TaskManager recovery (26) →
   orchestrator construction with production providers (28) → hotkey
   registration (30) → menu bar install → onboarding auto-open when
   `config` is fresh-default OR mic permission missing (34 owns the window).
   Boot failures surface as an alert + menu bar error badge, never a silent
   exit (DESIGN §12).
2. `MenuBarController` (AppKit `NSStatusItem`):
   - Icon per FSM state (observes orchestrator state stream), SF Symbols
     mapping exactly per DESIGN §8.1: idle `mic.slash`, listening `waveform`
     (animated via symbol effect), running `gearshape` + spinner badge,
     awaiting approval `exclamationmark.triangle.fill` (accent tint),
     speaking `speaker.wave.2`. Template images (dark/light safe).
   - Accessibility: `accessibilityLabel` per state (localized).
3. Menu content (rebuilt on open):
   - "Start Session ⌃⌥Space" / "End Session" (contextual);
   - Tasks section: bound + pending tasks as `TaskDigest` rows
     ("#2 handsfree — waiting for approval"), click → `startRequested(binding:)`
     into the orchestrator; terminal-unacknowledged rows show ✓/✗ and click
     acknowledges + opens HUD digest;
   - Project quick-pick submenu (registry entries, check on default);
   - "Settings…", "Run Onboarding…", "Quit Handsfree" (quit flow is 36 —
     until merged, plain terminate).
4. Localization scaffolding: `Sources/HandsfreeApp/Resources/en.lproj/
   Localizable.strings` + `ja.lproj/…`; helper `L10n.string(_:)`;
   **key-parity unit test** (DESIGN §14) lives here and covers all `.strings`
   files added by later UI issues.
5. Main-thread discipline: menu/status updates on MainActor; orchestrator
   remains off-main (state stream hops documented).
6. Testability: MenuBarController logic (icon mapping, menu model building)
   is a pure `MenuModel.build(state:, tasks:, projects:)` function unit-tested
   without AppKit; the AppKit layer renders the model.

## Acceptance Criteria

- [ ] `make app && open dist/Handsfree.app`: status item appears, no Dock
      icon, menu shows Start Session + Settings + Quit.
- [ ] Icon mapping unit test covers every FSM state.
- [ ] `MenuModel` tests: bound/pending/terminal task rows, empty registry,
      contextual start/end items.
- [ ] Strings: every user-visible literal in this PR goes through `L10n`;
      parity test green for en/ja.
- [ ] Boot failure path (corrupt config forced) shows alert + error badge
      (manual check documented).

## Validation

`swift test --filter MenuModelTests L10nParityTests`; manual launch checklist
in PR (screenshots: idle icon, menu open, ja locale run via
`-AppleLanguages '(ja)'`).

## Dependencies

05, 28 (and 30 for the hotkey label constant — soft: use config value).

## Non-goals

HUD, Settings tabs, notifications, quit-drain logic (36), custom icon design.

## Design References

DESIGN.md §8.1, §14, §9.2 (T10), §12; ADR-001.
