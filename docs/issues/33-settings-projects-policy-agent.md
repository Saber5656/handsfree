# Title

Settings tabs: Projects, Policy, Privacy, Agent

## Summary

Implement the remaining four Settings tabs over issue 32's scaffold:
project registry CRUD, approval policy switches with accurate risk copy,
privacy controls, and the codex agent configuration/preflight surface with
the T6 trust flow.

## Context

These tabs are the GUI for security-relevant configuration: tier policy
(ADR-004), transcript retention (T5), and the pinned codex binary (T6 —
confirmation is stored as `agent.codex_path_confirmed_path`; any path change
must reset trust). Every control maps to config keys (03) or actor APIs
(13/25/27) through 32's `ConfigBinding`.

## Scope

- Four tabs + view models. Scaffold/binding/General/Voice are 32. Menu-bar
  badging of preflight problems is issue 29/36 territory (the orchestrator
  surfaces preflight problems in its state snapshot; this issue only renders
  the Settings side).

## Detailed Requirements

1. **Projects tab** (registry 25): table (name, middle-truncated path, alias
   count, default ★); Add via `NSOpenPanel` (directories only) → name
   prefilled → inline `ProjectProblem` errors (non-git folder shows the
   exact problem + "Initialize git first" hint; NO auto-`git init`); edit
   sheet (name, aliases token field with auto-suggested space-split
   variants for ASCII names per 25, default toggle, model override); remove
   with confirmation; removing default clears it.
2. **Policy tab**:
   - `allow_tier3` toggle; caption (exact copy): "Allow full-access
     escalations (Tier 3). When off, actions needing access outside the
     project workspace are always denied. Network-only follow-ups such as
     git push remain available at Tier 2."
   - `tier3_screen_confirm` toggle (enabled only when allow_tier3): ON
     caption "Also requires a click on this Mac in addition to the spoken
     code — protects against voice spoofing."; turning OFF presents a
     confirmation alert restating the risk (DESIGN §5.4) and logs a
     `Log.approval` warning.
   - Read-only tier table (T0–T3 one-liners mirroring DESIGN §6.5).
3. **Privacy tab** (27): `store_transcripts` toggle; retention slider 0–365
   (0 labeled "session-only, never stored"); "Delete All Transcripts Now" →
   confirmation showing counts from `deletionSummary()` (non-mutating) →
   run `deleteAllNow()` → result toast; "Reveal Transcripts in Finder";
   static privacy summary (exact copy, four sentences): "Your voice is
   processed entirely on this Mac. Speech audio is never stored. Transcripts
   of your sessions are kept only on this Mac and deleted after the period
   above. What the coding agent sends to its own service is governed by
   your Codex account and configuration." (Issue 40 reconciles PRIVACY.md
   with this copy.)
4. **Agent tab** (13):
   - Resolved binary path (read-only) + "Change…" file picker. **T6 rule
     (tested)**: selecting a path performs ONE atomic config update that
     sets `agent.codex_path` = new path AND clears
     `agent.codex_path_confirmed_path` (trust reset). The preflight banner
     then shows `.binaryChanged(old,new)` with a "Trust This Binary" button
     which sets `codex_path_confirmed_path` = the banner's `new` path ONLY
     if the current resolved path still equals it (staleness check).
   - `.unsafeBinary(reason)` problems render as a blocking red status with
     the reason; the Trust button is DISABLED for unsafe binaries; dispatch
     stays blocked (13's gate).
   - Status lines: version (+ untested-version warning), auth state with a
     "How to log in" popover (`codex login` instructions — the app never
     runs it); "Re-check" button → `preflight.refresh()`.
   - Global model override, max-turn-seconds stepper (60–7200), max
     concurrent tasks (1–10).
5. All mutations via `ConfigBinding`/actor APIs; no direct file I/O in
   views; all strings via `L10n` (en/ja).

## Acceptance Criteria

- [ ] Projects VM tests: add valid git dir / add non-git → exact problem /
      alias suggestions / remove-default clears; manual matrix run.
- [ ] Policy VM tests: confirm-off alert flow; screen-confirm control inert
      when allow_tier3 off; copy matches Req 2 exactly.
- [ ] Privacy: `deletionSummary` → confirm → `deleteAllNow` round trip on
      fixture transcripts; retention writes config; reveal opens Finder
      (manual).
- [ ] Agent VM tests: path change resets trust atomically; Trust honors the
      staleness check; unsafeBinary disables Trust and shows blocking
      status; unauthenticated shows login guidance; re-check calls refresh.
- [ ] en/ja parity green; screenshots of all four tabs in both languages.

## Validation

`swift test --filter 'SettingsProjectsVMTests|SettingsPolicyVMTests|SettingsPrivacyVMTests|SettingsAgentVMTests'`;
manual checklist + screenshots.

## Dependencies

03, 13, 25, 27, 32.

## Non-goals

Voice-driven CRUD, per-project policy overrides (v2), keychain/cloud
settings, menu-bar badge rendering (29/36).

## Design References

DESIGN.md §8.3, §5.4, §6.5, §7.1–7.3, §9.2 (T5/T6), Appendix C.2 (amended);
ADR-004; issues 13/25/27 APIs.
