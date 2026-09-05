# Title

User and contributor documentation set

## Summary

Write the complete public documentation — `README.md`, `PRIVACY.md`,
`SECURITY.md`, `CONTRIBUTING.md`, `docs/QA_CHECKLIST.md` — accurate to the
shipped v1 behavior, and execute the first full manual QA pass.

## Context

An OSS voice tool that spawns coding agents lives on trust and first-run
success. Privacy claims must match pinned facts (on-device speech; network
= codex's own traffic + user-initiated Apple speech-asset downloads).
Drafting can start after 38/39; **closing requires 29–37 complete** because
the QA pass exercises the full app.

## Scope

- The five documents + README badges/links + the executed QA run record.
  English primary; README gets a compact Japanese section (DESIGN §14).

## Detailed Requirements

1. `README.md`: pitch (DESIGN §1.1) + condensed golden-path transcript
   (A.2; ja variant in the Japanese section); requirements box (macOS
   26.0+, Codex CLI installed & logged in via `codex login`, microphone);
   install (Releases download + signature verification pointer to
   SECURITY.md, or `make app` from source); first-run walkthrough; hotkey
   default ⌃⌥Space + rebind; enhanced-voice recommendation; architecture
   ASCII sketch (simplified §3.1) + user-language risk-tier table (§6.5);
   FAQ — including: "Does my voice leave my Mac?" (no; speech is processed
   on-device; the app's only network activity is Apple's speech-model
   download you trigger during setup — codex itself talks to its service
   under your account), "Why does push ask for a spoken code?", "Can I
   interrupt it while it speaks?" (v1: hotkey stops playback), "Which
   agents?" (Codex now; adapter interface for more); CI badge; license;
   link to docs/DESIGN.md. 日本語セクション: pitch、要件、A.1 短縮版、主要
   FAQ 対訳.
2. `PRIVACY.md`: data inventory table — voice audio (on-device, never
   stored), transcripts (local JSONL, retention setting, delete-all, exact
   location), speech model assets (downloaded from Apple when the user
   requests), codex traffic (governed by the user's codex account/config —
   link out), telemetry (none). Wording reconciled with the Settings
   privacy copy (33) and onboarding step 1 (34).
3. `SECURITY.md`: supported versions; reporting channel — GitHub private
   vulnerability reporting, described as the intended channel with a note
   that it is enabled at the branding/publication gate (issue 41 owns repo
   settings); artifact verification (`spctl`, checksums, notarization);
   condensed threat model linking DESIGN §9; the voice-approval design
   (nonce + Tier-3 click); out-of-scope (physical/keyboard access, codex
   upstream).
4. `CONTRIBUTING.md`: toolchain (CLT-only, no Xcode), `make` targets table,
   live-test convention (`*LiveTests`, issue 02) and how to run them, TCC
   note (always test mic flows from `make app` bundle, never bare
   `swift run`), module dependency rules (§3.1 — PRs violating import
   direction fail review), zero-dependency policy (ADR-010),
   docs-as-source-of-truth rule (behavior changes update DESIGN.md in the
   same PR), issue workflow (issues derive from docs/ISSUE_PLAN.md).
5. `docs/QA_CHECKLIST.md` (merging 38's hardware-measurement section):
   preconditions (TCC reset commands, fresh-config option, required
   hardware: built-in mic; optional: AirPods — items depending on optional
   hardware may be recorded as `blocked: <reason>`); scripted steps with
   expected earcons/speech for: golden path ja then en; approval matrix
   (T2 approve / T2 deny / T3 approve incl. click / T3 with screen-confirm
   off); cancel; idle timeout; background completion + notification bind;
   device switch mid-session; hotkey in a full-screen app; permission-denied
   recovery; hardware perf spot checks (hotkey→earcon stopwatch, idle CPU/
   RSS in Activity Monitor, real dispatch→first-narration). Each row:
   steps / expected / pass-fail-blocked.
6. Traceability: the PR includes a table mapping every README/PRIVACY claim
   → owning issue/test (prevents doc drift at launch).
7. Validation mechanics: README rendered-preview screenshot from the PR
   view; badge URLs checked with `curl -sI` (commands + output in PR);
   ja section approved by the maintainer (explicit PR comment is the
   close evidence).

## Acceptance Criteria

- [ ] All five documents complete; README preview screenshot + badge curl
      checks in PR.
- [ ] ja section: maintainer approval comment recorded.
- [ ] Traceability table complete (every claim → issue/test).
- [ ] QA checklist executed once fully on the dev machine; filled record
      committed as `docs/qa-runs/v0.1.0-dev.md` (optional-hardware rows may
      be `blocked` with reason).
- [ ] PRIVACY/SECURITY wording matches Settings (33) and onboarding (34)
      copy (diff note in PR).

## Validation

Rendered-doc review + executed QA record + traceability table in the PR.

## Dependencies

38, 39 (drafting). Closing additionally requires 29–37 complete (QA pass).

## Non-goals

Website, screencasts (placeholder noted), full localization beyond the
README ja section, CHANGELOG automation, enabling repo security settings
(41).

## Design References

DESIGN.md §1, §2, §6.5, §9, §14, Appendix A; ADR-003, ADR-010; ISSUE_PLAN
§6; research doc (asset-download network fact).
