# Title

App shell and menu bar controller

## Summary

Implement `HandsfreeApp`'s boot sequence and the `NSStatusItem` menu bar
presence: a state-truthful icon for every FSM state, the session/tasks menu,
localized UI strings scaffolding (en/ja), and resilient boot.

## Context

The menu bar is the only permanent UI (R1, DESIGN §8.1) and the T10
mic-state indicator. Boot must be able to show errors even when early steps
fail. Hotkey registration is a hard dependency (issue 30); onboarding UI is
owned by 34 and reached through a presenter seam defined here.

## Scope

- `@main` app delegate, boot wiring, MenuBarController + pure `MenuModel`,
  `L10n` + `.strings` scaffolding, `OnboardingPresenting` seam. Not: HUD
  (31), Settings content (32/33), onboarding UI (34), quit-drain (36).

## Detailed Requirements

1. Boot sequence (`applicationDidFinishLaunching`):
   1. **Install a minimal status item FIRST** (idle icon, menu with
      Settings…/Quit only) so later failures are visible.
   2. Config load (03) → logging init (04) → TaskManager recovery (26) →
      orchestrator construction with production providers (28) → hotkey
      registration (30) → menu upgrade to the full model → onboarding
      presenter check: if `config.general.onboarding_completed == false`,
      call `onboardingPresenter.presentIfNeeded()` (seam:
      `protocol OnboardingPresenting { func presentIfNeeded(); func present() }`
      — a no-op logger implementation ships here; 34 provides the real one).
   3. Any boot step failure → NSAlert with the error + menu bar **error
      badge** (exclamation overlay) + a "Boot problem…" menu item that
      reopens the alert; the app stays running with whatever subsystems are
      healthy (DESIGN §12).
2. Status icon — exhaustive FSM-state mapping (template images, SF Symbols;
   accessibility label key per row; T10 truth table):
   | FSM state | Symbol | Badge/animation |
   |---|---|---|
   | idle | `mic.slash` | — |
   | starting | `mic.slash` | pulse |
   | listening | `waveform` | animated |
   | interpreting | `waveform` | static |
   | confirming_dispatch | `waveform.badge.exclamationmark`? no — `waveform` | question badge |
   | dispatching | `gearshape` | spinner |
   | agent_running.narrating | `gearshape` + `speaker.wave.2` badge | spinner |
   | agent_running.listening_limited | `gearshape` + `waveform` badge | spinner |
   | awaiting_input | `waveform` | question badge |
   | awaiting_approval.* | `exclamationmark.triangle.fill` | accent tint |
   | speaking_result | `speaker.wave.2` | — |
   | ending | `mic.slash` | pulse |
   | error_recoverable | `waveform` | error badge |
   (Exact composite technique — base symbol + small badge overlay — is
   implementation detail; the table's semantics are the contract. Mic-open
   states MUST use waveform-family symbols; mic-closed states MUST NOT.)
3. Menu content (`MenuModel.build(input:) -> MenuModel` — pure, fully
   testable):
   ```swift
   struct MenuModelInput {
       let sessionState: SessionStateSnapshot   // from 28
       let hotkeyDisplay: String                // from 30's KeyMap glyphs
       let boundTaskID: TaskID?
       let tasks: [TaskDigest]
       let projects: [ProjectEntry]; let defaultProjectID: UUID?
       let bootProblem: String?
       let mode: OrchestratorMode
   }
   enum MenuAction {                             // rendered items dispatch these
       case startSession, endSession, bindTask(TaskID), acknowledge(TaskID)
       case setDefaultProject(UUID), openSettings, runOnboarding, quit
       case showBootProblem
   }
   ```
   Rows: contextual "Start Session ⌃⌥Space"/"End Session"; tasks section —
   pending/unacknowledged digests ("#2 handsfree — waiting for approval");
   click on awaiting/running rows → `bindTask`; click on
   terminal-unacknowledged rows → `acknowledge` (digest display in the HUD
   is issue 31's handler); project quick-pick submenu (check = default) →
   `setDefaultProject`; "Settings…", "Run Onboarding…" → presenter seam,
   "Quit Handsfree" (plain terminate until 36).
4. Localization: `en.lproj/Localizable.strings` + `ja.lproj/…` under
   `Sources/HandsfreeApp/Resources/`; `L10n.string(_:)` helper; key-parity
   unit test covering all `.strings` files (extends automatically as later
   issues add keys).
5. MainActor discipline: menu/status updates on MainActor; orchestrator
   state stream consumed via a MainActor task.

## Acceptance Criteria

- [ ] `make app && open dist/Handsfree.app`: status item appears, no Dock
      icon; menu shows Start Session + Settings + Quit; hotkey (30) toggles
      a session — manual checklist incl. the frontmost-app matrix from
      issue 30 (TextEdit/Finder, no Accessibility permission — screenshot of
      the Accessibility pane without Handsfree).
- [ ] Icon mapping unit test covers EVERY FSM state (exhaustive switch, no
      default).
- [ ] `MenuModel` tests: bound/pending/terminal rows and their actions,
      empty registry, contextual start/end, boot-problem row, demo mode
      label.
- [ ] Boot-failure path: corrupt config forced → minimal status item +
      alert + error badge (manual note + screenshot).
- [ ] Every user-visible literal goes through `L10n`; parity test green.

## Validation

`swift test --filter 'MenuModelTests|L10nParityTests'`; manual launch
checklist + screenshots (en and ja via `-AppleLanguages '(ja)'`).

## Dependencies

05, 28, 30.

## Non-goals

HUD (31), Settings tabs (32/33), onboarding UI (34), notifications (35),
quit-drain (36), custom icon design.

## Design References

DESIGN.md §8.1, §14, §9.2 (T10), §12; ADR-001; issue 30 (KeyMap glyphs).
