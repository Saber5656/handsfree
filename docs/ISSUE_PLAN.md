# Handsfree v1 — Issue Plan

- Derived from: [DESIGN.md](DESIGN.md) (source of truth for behavior)
- Issue drafts: `docs/issues/NN-short-title.md` (English; GitHub Issues are
  created from these files and are derived artifacts)
- Audience: implementation agents of modest capability — each issue is one
  coherent unit executable without guessing.

## 1. v1 completion statement

**v1 is complete when all 41 issues below are completed and validated.** At
that point:

1. The golden path (DESIGN §1.4, Appendix A) works end-to-end in Japanese and
   English on macOS 26 with a real microphone and a logged-in Codex CLI.
2. Every security acceptance criterion of DESIGN §9 has a passing automated
   test or a checked manual audit item (issue 37).
3. `make release` produces a signed, notarized, stapled `Handsfree.app` zip
   with checksums, publishable to GitHub Releases (issue 39).
4. CI runs build + full mock-based test suite on macos-26 runners (issues 02, 38).
5. README/PRIVACY/SECURITY/CONTRIBUTING/QA_CHECKLIST are complete (issue 40).

Anything discovered during implementation that is not covered below must become
a new issue derived from DESIGN.md — not silent scope in an existing PR.

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | issues/01-swiftpm-scaffold.md | SwiftPM package scaffold, targets, Makefile | 0 |
| 02 | issues/02-ci-workflow.md | CI workflow (build + test, macos-26) | 0 |
| 03 | issues/03-config-store.md | Config store: schema, atomic writes, permissions | 0 |
| 04 | issues/04-logging-redaction.md | Structured logging with secret redaction | 0 |
| 05 | issues/05-app-bundle-script.md | `.app` bundle assembly script + Info.plist/entitlements | 0 |
| 06 | issues/06-audio-engine-manager.md | Audio capture lifecycle (AVAudioEngine) | 1 |
| 07 | issues/07-stt-provider-protocol.md | STTProvider/VADProvider protocols + mocks | 1 |
| 08 | issues/08-apple-stt-provider.md | AppleSTTProvider (SpeechTranscriber + assets) | 1 |
| 09 | issues/09-vad-endpointing.md | VAD + utterance endpointing policy | 1 |
| 10 | issues/10-tts-provider.md | TTSProvider protocol + mock + AppleTTSProvider | 1 |
| 11 | issues/11-speech-output-arbiter.md | Half-duplex speech output arbiter + earcons | 1 |
| 12 | issues/12-agent-adapter-protocol.md | AgentAdapter protocol + AgentEvent model | 2 |
| 13 | issues/13-codex-preflight.md | Codex binary resolution, pinning, version/auth preflight | 2 |
| 14 | issues/14-process-runner.md | Child process runner (spawn/stream/cancel/timeout) | 2 |
| 15 | issues/15-jsonl-decoder.md | JSONL event decoder + recorded fixtures | 2 |
| 16 | issues/16-response-contract.md | Structured response contract + fallback chain | 2 |
| 17 | issues/17-fake-codex-fixture.md | FakeCodex scripted fixture executable | 2 |
| 18 | issues/18-codex-exec-adapter.md | CodexExecAdapter (compose 13–17, tier flags, resume) | 2 |
| 19 | issues/19-phrase-intent-matcher.md | Phrase dictionaries (ja/en) + IntentMatcher | 3 |
| 20 | issues/20-speech-text-sanitizer.md | Spoken-text sanitizer (TTS/notification boundary) | 3 |
| 21 | issues/21-approval-engine.md | Nonce approval engine | 3 |
| 22 | issues/22-session-fsm.md | Session state machine | 3 |
| 23 | issues/23-narration-policy.md | Narration policy engine | 3 |
| 24 | issues/24-prompt-scaffold.md | Prompt scaffold builder + templates | 3 |
| 25 | issues/25-project-registry.md | Project registry + voice name resolution | 3 |
| 26 | issues/26-task-manager.md | Task manager (background continuation, recovery) | 3 |
| 27 | issues/27-transcript-store.md | Transcript store + retention sweep | 3 |
| 28 | issues/28-dialogue-orchestrator.md | Dialogue orchestrator wiring + mock E2E | 3 |
| 29 | issues/29-app-shell-menu-bar.md | App shell + menu bar controller | 4 |
| 30 | issues/30-global-hotkey.md | Global hotkey manager + recorder control | 4 |
| 31 | issues/31-session-hud.md | Session HUD panel (incl. approval buttons) | 4 |
| 32 | issues/32-settings-general-voice.md | Settings window scaffold + General/Voice tabs | 4 |
| 33 | issues/33-settings-projects-policy-agent.md | Settings: Projects/Policy/Privacy/Agent tabs | 4 |
| 34 | issues/34-onboarding.md | Onboarding flow (permissions, assets, preflight, demo) | 4 |
| 35 | issues/35-notifications.md | Background task notifications | 4 |
| 36 | issues/36-app-lifecycle.md | Launch at login, quit-with-tasks drain, crash recovery hook | 4 |
| 37 | issues/37-security-hardening-tests.md | Security acceptance tests + audit (threats T1–T10) | 5 |
| 38 | issues/38-e2e-golden-path.md | E2E golden-path suite (mock speech + FakeCodex, ja/en) | 5 |
| 39 | issues/39-release-pipeline.md | Sign/notarize/release pipeline + workflow | 5 |
| 40 | issues/40-user-docs.md | User docs: README, PRIVACY, SECURITY, CONTRIBUTING, QA checklist | 5 |
| 41 | issues/41-branding-gate.md | Pre-release branding gate (name, bundle id freeze) | 5 |

## 3. Dependency table

`A ← B` means B must be completed before A starts. Issues not listed under a
wave gate depend only on their listed prerequisites.

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01 |
| 04 | 01 |
| 05 | 01 |
| 06 | 01 |
| 07 | 01 |
| 08 | 06, 07 |
| 09 | 06, 07 |
| 10 | 01 |
| 11 | 06, 07, 10 |
| 12 | 01 |
| 13 | 03, 12 |
| 14 | 12 |
| 15 | 12 |
| 16 | 12, 20 |
| 17 | 12, 15 |
| 18 | 13, 14, 15, 16, 17 |
| 19 | 01 |
| 20 | 01 |
| 21 | 19, 20 |
| 22 | 12, 19 |
| 23 | 12, 19, 20 |
| 24 | 01 |
| 25 | 03, 19 |
| 26 | 03, 12, 27 |
| 27 | 03, 04 |
| 28 | 07, 10, 11, 17, 18, 21, 22, 23, 24, 25, 26 |
| 29 | 05, 28 |
| 30 | 01 |
| 31 | 28 |
| 32 | 03, 08, 10, 30 |
| 33 | 03, 13, 25 |
| 34 | 08, 10, 13, 17, 25, 30 |
| 35 | 20, 26 |
| 36 | 26, 29 |
| 37 | 16, 18, 20, 21, 27 |
| 38 | 28 (uses 17) |
| 39 | 02, 05 |
| 40 | 38, 39 (content-final) |
| 41 | — (research-only; must land before first public release) |

Notes:

- 16 ← 20: the response contract applies the sanitizer's caps at parse time.
- 22 ← 12/19: the FSM consumes `AgentEvent` and intent types but no
  implementations (mock-driven tests).
- 30 has no UI dependencies; it is a standalone component consumed by 29/32/34.

## 4. Implementation waves

| Wave | Issues | Gate to next wave | Parallelism |
|---|---|---|---|
| 0 Foundation | 01–05 | `swift test` green in CI; `make app` produces launchable empty shell | 02–05 parallel after 01 |
| 1 Speech | 06–11 | live mic → transcription smoke (manual); mocks complete | fully parallel after 06/07 |
| 2 Agent | 12–18 | adapter integration tests green vs FakeCodex incl. blocked/needs_input/crash scenarios | 13–17 parallel after 12 |
| 3 Dialogue | 19–28 | mock E2E golden path (ja+en) green in CI | 19/20/24/27 first; 28 last |
| 4 App/UI | 29–36 | manual golden path with real mic + real codex on dev machine | 30 anytime; 31–35 after 28/29 |
| 5 Hardening/Release | 37–41 | v1 completion statement (§1) fully satisfied | 37/38/39 parallel; 40/41 last |

Waves 1 and 2 are mutually independent and may proceed in parallel after wave 0.

## 5. Coverage: DESIGN.md sections → issues

| DESIGN section | Issue(s) |
|---|---|
| §3.1–3.3 architecture, layout | 01, 05 |
| §4.1 audio lifecycle | 06 |
| §4.2 STT provider | 07, 08 |
| §4.3 VAD/endpointing | 09 |
| §4.4 TTS + arbiter | 10, 11 |
| §4.5 earcons | 11 |
| §5.1 session FSM | 22, 28 |
| §5.2 intents/matching | 19 |
| §5.3 narration | 23 |
| §5.4 approval engine | 21, 31 (T3 button) |
| §6.1 adapter protocol | 12 |
| §6.2 CodexExecAdapter | 13, 14, 15, 18 |
| §6.3 response contract | 16 |
| §6.4 prompt scaffold | 24 |
| §6.5 tier mapping | 18, 21, 37 |
| §6.6 task manager | 26, 35 |
| §7.1 project registry | 25, 33 |
| §7.2 config store | 03, 32, 33 |
| §7.3 transcript store | 27 |
| §8.1 menu bar | 29 |
| §8.2 HUD | 31 |
| §8.3 settings | 32, 33 |
| §8.4 onboarding | 34 |
| §8.5 notifications | 35 |
| §9 security model | 37 (+ embedded acceptance criteria in 03, 04, 13, 14, 16, 18, 20, 21, 27) |
| §10 build/release | 05, 39 |
| §11 testing strategy | 17, 38 (+ per-issue Validation) |
| §12 failure catalog | distributed: 06, 08, 10, 13, 14, 16, 18, 26, 27, 36 |
| §13 logging | 04 |
| §14 i18n | 19, 29, 32 (strings), 40 |
| §15 performance budgets | 06, 38 |
| §16 known unknowns | §8 below |
| Appendix A golden path | 38, 40 (QA checklist) |
| Appendix B phrases | 19 |
| Appendix C schemas | 03, 16 |

Every DESIGN section is owned by at least one issue; no v1 behavior lives only
in prose outside this table.

## 6. Whole-product validation strategy

1. **Per-issue Validation sections** are mandatory and must pass before an
   issue closes (unit/integration tests, or documented manual steps when
   hardware is involved).
2. **CI (issue 02)** runs `swift build` + all mock-based tests on macos-26 for
   every PR; live-audio and live-codex tests are excluded by test tags.
3. **FakeCodex (17) + mock speech providers (07/10)** are the deterministic
   backbone: the orchestrator E2E (28) and golden-path suite (38) run entirely
   offline in CI, covering happy path, needs_input, blocked/approve, blocked/deny,
   cancel, timeout, malformed output, and crash-recovery flows in ja and en.
4. **Security acceptance (37)**: executable tests for T1/T2/T3/T5/T6/T7 abuse
   cases; manual audit checklist for T8–T10 signed off per release.
5. **Manual QA (40)**: `docs/QA_CHECKLIST.md` scripted pass on real hardware
   (real mic, real codex, both languages) before any release tag.
6. **Release verification (39)**: notarization + `spctl -a` + fresh-machine
   Gatekeeper launch check documented in the release workflow.

## 7. Deferred to v2 (recorded, not planned here)

Wake word · barge-in/AEC · named multi-task voice switching · additional agent
adapters (Claude Code, opencode) · cloud STT/TTS opt-in · macOS 15 fallback
STT provider · pre-granted tier elevation · app-executed git verbs ·
auto-update (Sparkle) · Homebrew cask · MAS/App Sandbox investigation ·
UI locales beyond en/ja.

## 8. Known unknowns → potential new issues

| # | Unknown (DESIGN §16) | Trigger for new issue |
|---|---|---|
| 1 | SpeechDetector runtime behavior | If fallback endpointing is needed → follow-up in 09 |
| 2 | SpeechAnalyzer TCC requirements | If extra permission flow needed → follow-up in 34 |
| 3 | codex exit codes / resume-after-SIGINT semantics | If contract assumptions break → follow-up in 18 |
| 4 | `--output-schema` stability across codex versions | If enforcement regresses → follow-up in 16 |
| 5 | Default hotkey collisions | If ⌃⌥Space conflicts widely → config default change (03/30) |
| 6 | Notarization flakiness in CI | If blocking → local-release runbook issue (39) |
| 7 | ja/en code-switched STT accuracy | If unacceptable → custom-vocabulary issue after 08 |
| 8 | Final product name | Decided in 41; may spawn a rename sweep issue |
