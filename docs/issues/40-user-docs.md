# Title

User and contributor documentation set

## Summary

Write the complete public documentation: `README.md`, `PRIVACY.md`,
`SECURITY.md`, `CONTRIBUTING.md`, and `docs/QA_CHECKLIST.md` (the manual
release QA script) — accurate to the shipped v1 behavior.

## Context

An OSS voice tool that spawns coding agents lives or dies on trust and
first-run success. The docs must state the privacy model plainly (ADR-003),
the security model honestly (DESIGN §9), and the exact requirements
(macOS 26+, Codex CLI, mic).

## Scope

- The five documents + README badges/links. English primary; README gets a
  compact Japanese section (DESIGN §14).

## Detailed Requirements

1. `README.md`:
   - One-paragraph pitch mirroring DESIGN §1.1 + a golden-path example
     transcript (condensed A.2, plus the ja variant in the Japanese section).
   - Requirements box: macOS 26.0+, Codex CLI installed & logged in
     (`codex login`), microphone. Install: download from Releases (signature
     verification pointer to SECURITY.md) or build from source
     (`make app`).
   - First-run: onboarding walkthrough summary; hotkey default ⌃⌥Space and
     how to change; enhanced-voice recommendation.
   - How it works diagram (ASCII, from DESIGN §3.1 simplified) + the risk-tier
     table (DESIGN §6.5) in user language.
   - FAQ: "Does my voice leave my Mac?" (no — on-device; only codex's own
     traffic), "Why does push ask for a code?", "Can it interrupt while
     speaking?" (v1 no — hotkey stops playback), "Which agents?" (Codex now;
     adapter interface for more).
   - Status badge (CI), license, link to DESIGN.md for the deep dive.
   - 日本語セクション: pitch, requirements, ゴールデンパス例 (A.1 condensed),
     主要FAQの対訳.
2. `PRIVACY.md`: data inventory table (voice audio: processed on-device,
   never stored; transcripts: local JSONL, retention setting, delete-all;
   what codex sends upstream is governed by the user's codex account/config —
   link out; no telemetry of any kind). Must match implemented behavior —
   cross-check against 27's tests and onboarding copy (34 placeholder is
   replaced by final text here).
3. `SECURITY.md`: supported versions; how to report (GitHub private
   vulnerability reporting — enable in repo settings, note for maintainer);
   artifact verification (`spctl`, checksums, notarization); condensed threat
   model with link to DESIGN §9; the voice-approval design (nonce + Tier-3
   click) explained; out-of-scope items (physical/keyboard access, codex
   upstream).
4. `CONTRIBUTING.md`: toolchain (CLT-only, no Xcode required), `make` targets
   table, test tags (`live`) and how to run them, TCC note (use `make app`
   bundle for mic testing, never bare `swift run`), module dependency rules
   (DESIGN §3.1 — PRs violating the import rules fail review), zero-dependency
   policy (ADR-010), docs-as-source-of-truth rule (behavior changes update
   DESIGN.md in the same PR), issue workflow (issues derive from
   docs/ISSUE_PLAN.md).
5. `docs/QA_CHECKLIST.md`: scripted manual pass executed per release
   (merges 38's hardware-measurement section):
   preconditions (reset TCC, fresh config option); golden path ja then en
   (each step with expected earcon/speech); approval matrix (T2 approve, T2
   deny, T3 approve incl. click, T3 with screen-confirm off); cancel; idle
   timeout; background completion + notification bind; device switch
   (AirPods) mid-session; hotkey-in-fullscreen-app; permissions matrix
   (deny mic → recovery); perf spot checks (hotkey→earcon stopwatch, idle CPU
   in Activity Monitor). Each item: steps / expected / pass-fail column.
6. Consistency sweep: every claim in these docs must be traceable to an
   implemented issue; a review table in the PR maps README claims → issue
   numbers (prevents doc drift at launch).

## Acceptance Criteria

- [ ] All five documents complete; README renders correctly on GitHub
      (checked via preview), badges resolve.
- [ ] ja section reviewed for natural phrasing (owner review requested).
- [ ] PRIVACY/SECURITY claims cross-checked against tests (traceability table
      in PR).
- [ ] QA checklist executed once fully on the dev machine; filled sheet
      committed as `docs/qa-runs/v0.1.0-dev.md` (template proof).
- [ ] `L10n`-style spellcheck/link check pass (`lychee` or manual link click
      list in PR — no repo dependency added).

## Validation

Rendered-doc review + executed QA sheet + traceability table in the PR.

## Dependencies

38, 39 (behavior and release flow must be final enough to document).

## Non-goals

Website, screencasts/GIFs (placeholder noted), localized full docs beyond the
README ja section, CHANGELOG automation.

## Design References

DESIGN.md §1, §2, §6.5, §9, §14, Appendix A; ADR-003, ADR-010; ISSUE_PLAN §6.
