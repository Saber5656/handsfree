# Handsfree v1 — Issue Plan

- Derived from: [DESIGN.md](DESIGN.md) (source of truth for behavior)
- Issue drafts: `docs/issues/NN-short-title.md` (English; GitHub Issues are
  created from these files and are derived artifacts)
- Audience: implementation agents of modest capability — each issue is one
  coherent unit executable without guessing.
- Review status: every issue draft passed a per-issue Codex review plus a
  holistic cross-consistency review (2026-07-08); findings were fixed before
  this plan was finalized.

## 1. v1 completion statement

**v1 is complete when all 41 issues below are completed and validated.** At
that point:

1. The golden path (DESIGN §1.4, Appendix A) works end-to-end in Japanese
   and English on macOS 26 with a real microphone and a logged-in Codex CLI.
2. Every security acceptance criterion of DESIGN §9 has a passing automated
   test or a checked audit item (issue 37; release-time evidence via 39/40).
3. `release.sh` produces a signed, notarized, stapled `Handsfree.app` zip
   with checksums, publishable as a draft GitHub Release (issue 39).
4. CI runs build + the full mock-based suite on macos-26 (issues 02, 38).
5. README/PRIVACY/SECURITY/CONTRIBUTING/QA_CHECKLIST are complete and a full
   manual QA pass is recorded (issue 40).
6. The branding/publication gate has been signed off by the maintainer
   (issue 41).

Anything discovered during implementation that is not covered below must
become a new issue derived from DESIGN.md — not silent scope in an existing
PR.

## 2. Issue list (recommended execution order)

Titles are verbatim copies of each issue file's Title section.

| # | File | Title | Wave |
|---|---|---|---|
| 01 | issues/01-swiftpm-scaffold.md | SwiftPM package scaffold: targets, module boundaries, Makefile, formatting | 0 |
| 03 | issues/03-config-store.md | Config store: versioned schema, atomic writes, strict permissions | 0 |
| 04 | issues/04-logging-redaction.md | Structured logging facade with secret redaction and diagnostic snapshot | 0 |
| 05 | issues/05-app-bundle-script.md | `.app` bundle assembly: make-app.sh, Info.plist template, entitlements, icon | 0 |
| 02 | issues/02-ci-workflow.md | CI workflow: build, test, lint on macos-26 with SHA-pinned actions | 0 |
| 06 | issues/06-audio-engine-manager.md | AudioEngineManager: microphone capture lifecycle with strict session binding | 1 |
| 07 | issues/07-stt-provider-protocol.md | STTProvider/VADProvider protocols, result types, and scriptable mocks | 1 |
| 08 | issues/08-apple-stt-provider.md | AppleSTTProvider: SpeechTranscriber streaming STT with asset management | 1 |
| 09 | issues/09-vad-endpointing.md | VAD integration and utterance endpointing policy | 1 |
| 10 | issues/10-tts-provider.md | TTSProvider protocol, AppleTTSProvider, and MockTTSProvider | 1 |
| 11 | issues/11-speech-output-arbiter.md | SpeechOutputArbiter: half-duplex gate, priority queue, earcons | 1 |
| 12 | issues/12-agent-adapter-protocol.md | AgentAdapter protocol and AgentEvent domain model | 2 |
| 14 | issues/14-process-runner.md | ProcessRunner: child process spawn/stream/cancel/timeout with process groups | 2 |
| 13 | issues/13-codex-preflight.md | CodexPreflight: binary resolution with pinning, version and auth checks | 2 |
| 15 | issues/15-jsonl-decoder.md | Codex JSONL event decoder with recorded fixtures | 2 |
| 16 | issues/16-response-contract.md | Structured response contract: schema resource, parser, fallback chain | 2 |
| 17 | issues/17-fake-codex-fixture.md | FakeCodex: scripted codex-CLI stand-in for tests, CI, and onboarding demo | 2 |
| 18 | issues/18-codex-exec-adapter.md | CodexExecAdapter: turn execution with tier flags, resume, and cancellation | 2 |
| 19 | issues/19-phrase-intent-matcher.md | Phrase dictionaries (ja/en) and IntentMatcher | 3 |
| 20 | issues/20-speech-text-sanitizer.md | SpeechTextSanitizer: the validation layer for everything spoken or notified | 3 |
| 24 | issues/24-prompt-scaffold.md | PromptScaffoldBuilder: versioned prompt templates for every turn kind | 3 |
| 27 | issues/27-transcript-store.md | TranscriptStore: session JSONL records, task index primitives, retention | 3 |
| 21 | issues/21-approval-engine.md | ApprovalEngine: nonce generation, echo verification, tiered decision flow | 3 |
| 22 | issues/22-session-fsm.md | Session state machine: pure reducer with exhaustive transition table | 3 |
| 23 | issues/23-narration-policy.md | NarrationPolicy: verbosity-tiered, throttled progress narration | 3 |
| 25 | issues/25-project-registry.md | ProjectRegistry: validated project entries and voice name resolution | 3 |
| 26 | issues/26-task-manager.md | TaskManager: task lifecycle, background continuation, pending queue, recovery | 3 |
| 28 | issues/28-dialogue-orchestrator.md | DialogueOrchestrator: wire FSM, speech, approval, agent, tasks — mock E2E | 3 |
| 30 | issues/30-global-hotkey.md | Global hotkey manager and shortcut recorder control | 4 |
| 29 | issues/29-app-shell-menu-bar.md | App shell and menu bar controller | 4 |
| 31 | issues/31-session-hud.md | Session HUD: floating non-activating panel with approval controls | 4 |
| 32 | issues/32-settings-general-voice.md | Settings window scaffold + General and Voice tabs | 4 |
| 33 | issues/33-settings-projects-policy-agent.md | Settings tabs: Projects, Policy, Privacy, Agent | 4 |
| 34 | issues/34-onboarding.md | Onboarding flow: permissions, speech assets, codex preflight, demo session | 4 |
| 35 | issues/35-notifications.md | Background task notifications | 4 |
| 36 | issues/36-app-lifecycle.md | App lifecycle: launch at login, single instance, quit-with-tasks drain | 4 |
| 37 | issues/37-security-hardening-tests.md | Security acceptance tests and v1 audit checklist (threats T1–T10) | 5 |
| 38 | issues/38-e2e-golden-path.md | E2E golden-path suite and performance budget checks | 5 |
| 39 | issues/39-release-pipeline.md | Release pipeline: Developer ID signing, notarization, GitHub Release workflow | 5 |
| 40 | issues/40-user-docs.md | User and contributor documentation set | 5 |
| 41 | issues/41-branding-gate.md | Pre-release branding gate: name decision, identifier freeze, publication scan | 5 |

## 3. Dependency table

`A ← B` means B must be completed before A starts. This table matches each
issue file's Dependencies section 1:1 (hard prerequisites only; soft
coordination notes live in the issue texts).

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01, 05 |
| 03 | 01 |
| 04 | 01 |
| 05 | 01 |
| 06 | 01 |
| 07 | 01, 06 |
| 08 | 05, 06, 07 |
| 09 | 06, 07, 08 |
| 10 | 01, 07 |
| 11 | 06, 07, 10 |
| 12 | 01 |
| 13 | 03, 12, 14 |
| 14 | 01, 12 |
| 15 | 12 |
| 16 | 12 |
| 17 | 12, 15 |
| 18 | 13, 14, 15, 16, 17 |
| 19 | 01 |
| 20 | 01 |
| 21 | 19, 20 |
| 22 | 12, 19 |
| 23 | 10, 12, 19, 20 |
| 24 | 01 |
| 25 | 03, 19 |
| 26 | 03, 12, 14, 27 |
| 27 | 03, 04 |
| 28 | 07, 10, 11, 17, 18, 21, 22, 23, 24, 25, 26, 27 |
| 29 | 05, 28, 30 |
| 30 | 01 |
| 31 | 28, 29 |
| 32 | 03, 04, 08, 10, 11, 30 |
| 33 | 03, 13, 25, 27, 32 |
| 34 | 05, 08, 10, 13, 17, 25, 28, 30, 31 |
| 35 | 20, 26, 28 |
| 36 | 26, 29, 32 |
| 37 | 02, 16, 18, 20, 21, 27, 28 |
| 38 | 28 |
| 39 | 02, 05 |
| 40 | 38, 39 (drafting) — closing additionally requires 29–37 complete |
| 41 | 35, 39, 40 |

## 4. Implementation waves

| Wave | Issues (in-wave order) | Gate to next wave | Parallelism notes |
|---|---|---|---|
| 0 Foundation | 01 → {03, 04, 05} → 02 | CI green (build+test+lint+unsigned app smoke); `make app` launches the placeholder | 03/04/05 parallel after 01; 02 needs 05 |
| 1 Speech | 06 → 07 → {08, 10} → {09, 11} | live mic→transcription smoke (manual); all mocks complete | pairs parallel as shown |
| 2 Agent | 12 → {14, 15} → {13, 16, 17} → 18 | adapter integration green vs FakeCodex (all scenarios incl. escalation matrix) | 15/16 parallel; 13 needs 14 |
| 3 Dialogue | {19, 20, 24, 27} → {21, 22, 25} → {23, 26} → 28 | mock E2E golden path (ja+en) green in CI | wave 1 and wave 2 may themselves run in parallel before this wave |
| 4 App/UI | 30 → 29 → 31 → 32 → {33, 35} → {34, 36} | manual golden path with real mic + real codex on the dev machine | 30 can start any time after 01 |
| 5 Hardening/Release | {37, 38, 39} → 40 → 41 | v1 completion statement (§1) fully satisfied | 37/38/39 parallel |

Waves 1 and 2 are mutually independent and may proceed in parallel after
wave 0.

## 5. Coverage: DESIGN.md sections → issues

| DESIGN section | Issue(s) |
|---|---|
| §1 product definition, golden path | 28, 38, 40 |
| §2 scope / non-goals | 40 (docs); enforced by this plan's totality |
| §3.1–3.3 architecture, layout | 01, 05 |
| §4.1 audio lifecycle | 06 |
| §4.2 STT provider | 07, 08 |
| §4.3 VAD/endpointing | 09 |
| §4.4 TTS + arbiter | 10, 11 |
| §4.5 earcons | 11 |
| §5.1 session FSM | 22, 28 |
| §5.2 intents/matching | 19 |
| §5.3 narration | 23 |
| §5.4 approval engine | 21, 31 (T3 buttons) |
| §6.1 adapter protocol | 12 |
| §6.2 CodexExecAdapter | 13, 14, 15, 18 |
| §6.3 response contract | 16 |
| §6.4 prompt scaffold | 24 |
| §6.5 tier mapping | 18, 21, 28, 37 |
| §6.6 task manager | 26, 35 |
| §7.1 project registry | 25, 33 |
| §7.2 config store | 03, 32, 33 |
| §7.3 transcript store | 27 |
| §8.1 menu bar | 29 |
| §8.2 HUD | 31 |
| §8.3 settings | 32, 33 |
| §8.4 onboarding | 34 |
| §8.5 notifications | 35 |
| §9 security model | 37 (+ embedded acceptance criteria in 03, 04, 13, 14, 16, 18, 20, 21, 26, 27, 28) |
| §10 build/release | 02, 05, 39 |
| §11 testing strategy | 02 (live convention), 07, 10, 17, 38 (+ per-issue Validation) |
| §12 failure catalog | distributed: 06, 08, 10, 13, 14, 16, 18, 26, 27, 29, 36 |
| §13 logging | 04 |
| §14 i18n | 19 (spoken), 29 (UI strings), 40 (docs) |
| §15 performance budgets | 06, 28, 38 (+ QA doc in 40) |
| §16 known unknowns | see §8 below (owners: 08, 09, 13/16/18, 30/32, 39, 41) |
| Appendix A golden path | 28, 38, 40 |
| Appendix B phrases | 19 |
| Appendix C schemas | 03, 16 |

Every DESIGN section is owned by at least one issue; no v1 behavior lives
only in prose outside this table.

## 6. Whole-product validation strategy

1. **Per-issue Validation sections** are mandatory and must pass before an
   issue closes. Test suites requiring hardware/assets/codex login use the
   `*LiveTests` naming convention (issue 02) and are excluded from CI.
2. **CI (02)** runs `swift build`, `swift test --skip 'LiveTests$'`,
   `make lint`, the unsigned `make app` smoke, ad-hoc sign check (39
   amendment), and `scripts/security-checks.sh` (37 amendment) on macos-26
   for every PR.
3. **FakeCodex (17) + mock speech providers (07/10)** are the deterministic
   backbone: the orchestrator E2E (28) and golden-path suite (38) run
   offline in CI, covering happy path, needs_input, blocked/approve (both
   tiers), blocked/deny, cancel, timeout, malformed output, crash recovery,
   and locale variants.
4. **Security acceptance (37)**: executable abuse tests for
   T1/T2/T3/T5/T6/T7 (+ the T10 mic property); static tripwires in CI;
   audit checklist for T8–T10 with release-time evidence via 39/40.
5. **Manual QA (40)**: scripted `docs/QA_CHECKLIST.md` pass on real hardware
   (real mic, real codex, ja+en) recorded per release.
6. **Release verification (39)**: notarization, `spctl`, stapling, fresh-VM
   Gatekeeper check, checksums — draft release; publishing is a human gate
   (merge ≠ release).

## 7. Deferred to v2 (recorded, not planned here)

Wake word · barge-in/AEC · named multi-task voice switching · additional
agent adapters (Claude Code, opencode) · cloud STT/TTS opt-in · macOS 15
fallback STT provider · pre-granted tier elevation · app-executed git verbs ·
auto-update (Sparkle) · Homebrew cask · MAS/App Sandbox investigation ·
UI locales beyond en/ja · SLSA provenance.

## 8. Known unknowns → owners

| # | Unknown (DESIGN §16) | Owner / trigger for new issues |
|---|---|---|
| 1 | SpeechDetector runtime behavior | 09 closes it (empirical gate decides the default VAD) |
| 2 | SpeechAnalyzer TCC requirements | 08 closes it (pins prompts; 34 consumes the answer) |
| 3 | codex exit codes / resume-after-SIGINT semantics | 18 records evidence; research doc updated |
| 4 | `--output-schema` stability across codex versions | 16's fallback chain + contract tests; new issue on breakage |
| 5 | Default hotkey collisions (⌃⌥Space) | 30 documents fallback; 32 makes rebinding first-class; config default change if reports accumulate |
| 6 | Notarization flakiness in CI | 39's local-release runbook |
| 7 | ja/en code-switched STT accuracy | evaluated during 08/38 QA; custom-vocabulary issue if unacceptable |
| 8 | Final product name | 41 (maintainer decision gate) |
