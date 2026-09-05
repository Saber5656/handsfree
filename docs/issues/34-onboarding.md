# Title

Onboarding flow: permissions, speech assets, codex preflight, demo session

## Summary

Implement the 7-step onboarding of DESIGN §8.4 as a standalone window,
ending with a scripted hands-free demo session against the bundled
FakeCodex — plus the `OnboardingPresenting` implementation for issue 29's
seam and the make-app.sh amendment that bundles `fake-codex`.

## Context

Onboarding carries the privacy story (on-device speech; network = codex
traffic + user-initiated Apple asset downloads — amended DESIGN §8.4),
collects TCC permissions with correct bundle attribution, repairs the two
research-confirmed out-of-box gaps (missing en assets, compact-only
voices), and verifies codex before the first real task. The demo must never
touch the real codex path pinning (T6) or the user's projects.

## Scope

- Onboarding window + step flow + demo-session mode + `OnboardingPresenting`
  impl + make-app.sh amendment (05). Uses 08/10/13/17/25/28/30/31 APIs.

## Detailed Requirements

1. Container: standalone `NSWindow` (640×480, non-resizable, regular level;
   NOT an AppKit sheet — there is no parent window in an LSUIElement app),
   presented by the `OnboardingPresenting` implementation registered into
   issue 29's seam; auto-presents on boot when
   `general.onboarding_completed == false`; re-runnable from the menu.
   Linear steps with Back/Continue + progress dots; completion sets the
   config flag.
2. Steps (exact order; each step catches its component errors and renders
   retry guidance — onboarding never crashes the app):
   1. **Welcome/privacy** — copy verbatim from DESIGN §8.4 step 1 (amended:
      mentions user-initiated speech-asset downloads).
   2. **Microphone permission** — explain, then trigger TCC via a short
      `AudioEngineManager.start()/stop()` probe; denied → "Open System
      Settings → Privacy → Microphone" + re-check button; Continue disabled
      while denied (Quit-onboarding allowed). Speech-recognition permission
      is requested here IFF issue 08's pinned findings say the API demands
      it (this issue consumes 08's documented answer; the manual oracle is
      conditional accordingly).
   3. **STT assets** — language picker writes `general.locale_mode`;
      asset rows per locale with the mapping: mode `ja` → ja-JP required;
      `en` → en-US required; `auto` → BOTH rows shown, at least one
      installed required to continue (the session locale rule stays
      DESIGN §5.x: auto resolves at session start). Download buttons drive
      `prepare()` with `preparationProgress`.
   4. **TTS voices** — current voice + quality label; compact-only state →
      System Settings guidance reusing 32's fallback pattern; preview
      button. Skippable.
   5. **Codex preflight** — render found/version/auth with fix guidance
      (`codex login` copy). Skippable with warning (dispatch remains gated
      by preflight anyway).
   6. **First project** — folder picker → registry add (25) with inline
      problems; sets default. Skippable if registry non-empty.
   7. **Hotkey + demo** — show current hotkey (rebind link → Settings);
      "Try it now" runs a REAL orchestrator session in **demo mode**:
      - `orchestrator.mode = .demo` for the session; HUD shows the Demo
        badge (31);
      - the adapter is constructed with a **session-scoped codexPath
        override** pointing at the bundled `fake-codex`
        (`Bundle.main.bundleURL/Contents/MacOS/fake-codex`) and
        `FAKE_CODEX_SCENARIO=demo-ja|demo-en` by session locale — this
        override is in-memory only: it is NEVER written to
        `agent.codex_path`/`codex_path_confirmed_path` and never passes
        through CodexPreflight pinning (T6 assertion in tests);
      - demo project = a unique temp directory created for the demo,
        `git init`-ed before dispatch (projects must be git repos), passed
        via `-C`, NEVER added to the ProjectRegistry; removed at demo end
        (retained with a log line if removal fails).
      The user speaks a suggested phrase, hears the scripted summary,
      practices 「終了」.
3. `scripts/make-app.sh` amendment (05): build the `fake-codex` product and
   copy it to `Contents/MacOS/fake-codex` in both dev and release bundles
   (39's sign.sh already discovers nested executables).
4. Failure resilience: per-step typed error → inline retry UI (asset
   download no-network, preflight missing binary, mic denied, demo spawn
   failure).

## Acceptance Criteria

- [ ] Step-flow VM tests: gating rules (mic blocks, per-mode asset
      requirements incl. `auto`, skip rules), completion flag, re-run.
- [ ] Demo E2E test (orchestrator + mock speech + real adapter +
      fake-codex `demo-en`): demo badge state set; user registry and
      `agent.codex_path*` config keys UNCHANGED after the demo (T6
      assertions); scratch dir created+git-inited+cleaned; scripted summary
      spoken.
- [ ] make-app.sh bundles fake-codex; `codesign --verify --strict` still
      passes on the ad-hoc bundle.
- [ ] Manual full pass from a reset state (`tccutil reset Microphone
      <bundle-id>`, fresh config): all 7 steps; mic TCC prompt attributes to
      "Handsfree"; speech-recognition prompt appears/absent exactly per
      issue 08's pinned finding; ja demo heard. Checklist + screenshots.
- [ ] Denied-mic recovery path works after granting in System Settings.

## Validation

`swift test --filter 'OnboardingFlowTests|DemoModeE2ETests'`; manual
reset-state checklist in PR.

## Dependencies

05, 08, 10, 13, 17, 25, 28, 30, 31.

## Non-goals

Account creation, codex install/login automation, telemetry, wake-word
teaching.

## Design References

DESIGN.md §8.4 (amended), §9.2 (T6), §9.3, §12; research doc (asset/voice
gaps); ADR-003; issue 28 (mode API), 31 (demo badge).
