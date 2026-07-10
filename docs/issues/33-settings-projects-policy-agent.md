# Title

Settings tabs: Projects, Policy, Privacy, Agent

## Summary

Implement the remaining four Settings tabs over the scaffold from issue 32:
project registry CRUD, approval policy switches with plain-language risk
explanations, privacy controls, and codex agent configuration/preflight
surface.

## Context

These tabs are the GUI for security-relevant configuration: tier policy
(ADR-004), transcript retention (threat T5), and the pinned codex binary
(threat T6). Every control maps 1:1 to config keys (03) or actor APIs
(25/27/13).

## Scope

- Four tabs + their view models. Scaffold, binding helper, General/Voice are 32.

## Detailed Requirements

1. **Projects tab** (registry 25):
   - Table: name, path (middle-truncated), aliases count, default ★.
   - Add: folder picker (`NSOpenPanel`, directories only) → name prefilled
     from folder name → inline validation errors from
     `ProjectRegistry.add` (non-git folder shows the exact problem +
     "Initialize git first" hint; no auto-`git init`).
   - Edit sheet: name, aliases (token field; auto-suggest space-split variants
     for ASCII names per 25's doc note), default toggle, model override
     (free text, placeholder "codex default").
   - Remove with confirmation; removing default clears it.
2. **Policy tab**:
   - `allow_tier3` toggle: title "Allow full-access escalations (Tier 3)",
     caption explaining what danger-full-access means and that denying it
     limits some follow-ups (push outside workspace etc. still fine at T2).
   - `tier3_screen_confirm` toggle (enabled only when allow_tier3):
     ON caption "Requires a click on this Mac in addition to the spoken code —
     protects against voice spoofing"; turning OFF presents a confirmation
     alert restating the risk (DESIGN §5.4) and logs the change to transcripts
     (`error`-type record? No — add `config_change` record type in 27? Keep
     scope: log via Log.approval warning; transcript schema unchanged).
   - Read-only tier table (T0–T3 with one-line meanings) mirroring DESIGN §6.5
     so users see the model.
3. **Privacy tab** (27):
   - `store_transcripts` toggle + retention slider 0–365 days (0 labeled
     "session-only, never stored");
   - "Delete All Transcripts Now" → confirmation with file count from
     `deleteAllNow()` dry info → result toast;
   - "Reveal Transcripts in Finder" (`NSWorkspace.activateFileViewerSelecting`);
   - static privacy summary text (matches PRIVACY.md tone; what never leaves
     the machine).
4. **Agent tab** (13):
   - Detected binary path (read-only field) + "Change…" (file picker) writing
     `agent.codex_path` → triggers preflight refresh;
   - **Binary-changed confirmation UI**: when preflight reports
     `.binaryChanged(old:new:)`, show a warning banner with both paths and a
     "Trust new path" button setting `agent.codex_path_confirmed=true`
     (T6 flow; dispatch stays blocked until confirmed — banner also appears
     via menu bar error badge, 29);
   - Status lines: version (+ "untested version" warning when out of range),
     auth state with "How to log in" popover (`codex login` instructions —
     the app never runs it);
   - "Re-check" button (preflight.refresh());
   - Model override (global), max turn seconds (60–7200 stepper),
     max concurrent tasks (1–10 stepper).
5. All mutations through `ConfigBinding`/actor APIs; no direct file I/O in
   views. All strings via `L10n` (en/ja).

## Acceptance Criteria

- [ ] Projects: add(valid git dir)/add(non-git → exact error)/edit
      aliases/remove default — manual matrix + registry state assertions via
      view-model tests.
- [ ] Policy: tier3 confirm-off alert flow; disabled-state logic
      (screen_confirm control inert when allow_tier3 off) — view-model tests.
- [ ] Privacy: delete-all round trip on fixture transcripts; retention slider
      writes config; reveal opens Finder (manual).
- [ ] Agent: binaryChanged banner appears on a simulated path change and
      clears after Trust (view-model test with preflight stub + manual run);
      unauthenticated state shows login guidance.
- [ ] en/ja parity test green; screenshots of all four tabs in both languages.

## Validation

`swift test --filter SettingsProjectsVMTests SettingsPolicyVMTests
SettingsPrivacyVMTests SettingsAgentVMTests`; manual checklist + screenshots.

## Dependencies

03, 13, 25, 27, 32.

## Non-goals

Voice-driven CRUD, per-project policy overrides (v2), keychain/cloud settings.

## Design References

DESIGN.md §8.3, §5.4, §6.5, §7.1–7.3, §9.2 (T5/T6); ADR-004.
