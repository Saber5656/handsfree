# Title

Onboarding flow: permissions, speech assets, codex preflight, demo session

## Summary

Implement the 7-step onboarding window of DESIGN §8.4, ending with a scripted
hands-free demo session against a bundled FakeCodex — the user's risk-free
first loop.

## Context

Onboarding carries the privacy story (what stays on device), collects the
mic/speech permissions with correct TCC attribution, repairs the two
research-confirmed out-of-box gaps (missing en STT assets, compact-only TTS
voices), and verifies codex before the first real task.

## Scope

- Onboarding window + step flow + demo-session mode + make-app.sh amendment to
  bundle `fake-codex`. Uses existing components (08/10/13/25/30/17/28).

## Detailed Requirements

1. Window: 640×480 sheet-style, linear steps with Back/Continue, re-runnable
   from the menu ("Run Onboarding…"). Progress dots. Steps may be skipped
   only where noted; completion state stored in config
   (`general.onboarding_completed: Bool` — add key, update DESIGN C.2).
2. Steps (exact order, DESIGN §8.4):
   1. **Welcome/privacy**: static copy — speech processed on-device; the only
      network traffic is codex's own under the user's account; transcripts
      local with retention link to Settings. (Copy reviewed against
      PRIVACY.md in issue 40 — placeholder constant now.)
   2. **Microphone permission**: explain → request via a short
      `AudioEngineManager.start()/stop()` probe (triggers TCC); denied state
      shows "Open System Settings → Privacy → Microphone" deep link +
      re-check button. Speech-recognition permission is requested here too if
      08's findings require it (this issue consumes 08's documented answer).
      Not skippable while denied (Continue disabled; Quit-onboarding allowed).
   3. **STT assets**: per selected language mode (config from a language
      picker on this step): availability + Download with progress (08).
      Skippable only if at least one locale is `.available`.
   4. **TTS voices**: current voice + quality label; if compact-only,
      System Settings guidance (32's fallback pattern) + preview button.
      Skippable.
   5. **Codex preflight**: run 13; render found/version/auth states with fix
      guidance (`codex login` copy). Skippable with warning ("you can finish
      setup in Settings → Agent"; dispatch stays gated by preflight anyway).
   6. **First project**: folder picker → registry add (25) with inline
      validation; set as default. Skippable if registry non-empty.
   7. **Hotkey + demo**: show current hotkey (rebind link → Settings);
      "Try it now" starts a REAL orchestrator session in **demo mode**:
      adapter's codexPath pointed at the bundled `fake-codex` with
      `FAKE_CODEX_SCENARIO=demo-ja|demo-en` (by session locale) and project =
      a temp scratch git dir created for the demo (never the user's project).
      The user speaks a suggested phrase, hears the scripted summary,
      practices 「終了」. A "Demo" badge shows on the HUD during demo mode
      (visible distinction is a MUST — orchestrator gains a
      `mode: .normal|.demo` flag surfaced to HUD/menu).
3. `scripts/make-app.sh` amendment (05): build + copy `fake-codex` into
   `Contents/MacOS/fake-codex` (both dev and release bundles; ~small binary).
   Demo mode resolves it relative to the main executable path.
4. Auto-open on first launch (29 hook) when `onboarding_completed=false`.
5. Failure resilience: every step catches its component errors and renders
   retry guidance; onboarding never crashes the app (DESIGN §12).

## Acceptance Criteria

- [ ] Step-flow view-model tests: gating rules (mic step blocks, skip rules),
      completion flag persistence, re-run behavior.
- [ ] Demo mode: E2E test at orchestrator level (mock speech + real adapter +
      fake-codex `demo-en`) asserting demo badge state, scratch-dir isolation
      (user registry untouched), and scripted summary spoken.
- [ ] Manual full pass on the dev machine from a reset state
      (`tccutil reset Microphone <bundle-id>` + fresh config): all 7 steps,
      both TCC prompts correctly attributed to "Handsfree", ja demo heard.
      Checklist + screenshots in PR.
- [ ] make-app.sh bundles fake-codex; `codesign --verify --strict` still
      passes (nested binary signed in 39's flow; ad-hoc here).
- [ ] Denied-mic path renders recovery UI and re-check works after granting.

## Validation

`swift test --filter OnboardingFlowTests DemoModeE2ETests`; manual reset-state
checklist in PR.

## Dependencies

08, 10, 13, 17, 25, 30 (+ amends 05; uses 28).

## Non-goals

Account creation of any kind, codex installation/login automation, analytics
about onboarding completion (no telemetry).

## Design References

DESIGN.md §8.4, §9.3, §12; research doc (assets/voices gaps); ADR-003.
