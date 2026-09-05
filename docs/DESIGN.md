# Handsfree — v1 Design Document

- Status: Draft for review (requirements confirmed with the product owner, 2026-07-08)
- Scope: complete v1 architecture and behavior; implementation is decomposed in
  [ISSUE_PLAN.md](ISSUE_PLAN.md) and `docs/issues/*.md`
- Canonical: this file is the source of truth for product behavior. GitHub
  Issues are derived artifacts.

---

## 1. Product definition

### 1.1 One-liner

**Handsfree lets you drive coding agents end-to-end with your voice on macOS** —
start tasks, hear progress, answer the agent's questions, approve risky actions,
and iterate, without touching the keyboard.

### 1.2 Target user and positioning

- A developer on macOS who delegates implementation work to CLI coding agents
  (Codex CLI first) and wants to keep steering them while their hands/eyes are
  busy, away from the desk, or resting (RSI).
- Not a dictation tool (macOS already has one), not a general voice assistant.
  The entire dialogue model is specialized for the *task lifecycle of a coding
  agent*: dispatch → progress → clarification → approval → result → follow-up.

### 1.3 Confirmed requirements (product owner decisions, 2026-07-08)

| # | Decision | Detail |
|---|---|---|
| R1 | Form factor | macOS resident menu bar app, global activation |
| R2 | Agent integration | `AgentAdapter` interface; v1 implements Codex CLI headless (`codex exec`) only |
| R3 | Speech processing | STT/TTS behind provider interfaces; **local by default**, cloud opt-in later; no API keys in v1 |
| R4 | Core experience | Full voice dialogue loop incl. voice approvals and follow-ups |
| R5 | Tech stack | Swift native (SwiftUI/AppKit + Speech.framework + AVAudioEngine) |
| R6 | Session start | Global hotkey → continuous conversation via VAD; mic fully off outside sessions; wake word deferred to v2 |
| R7 | Approval policy | Risk-tiered: workspace edits voice-approvable; irreversible/external actions require nonce echo, highest tier additionally on-screen confirm by default |
| R8 | Concurrency | One foreground conversation; dispatched tasks continue in background with voice completion announcements; voice task switching by name deferred to v2 |
| R9 | Languages | Japanese + English from v1, dictionary-driven phrase design; UI/docs in English |
| R10 | Result narration | Agent self-summarization: orchestrator enforces a structured final response (voice summary) via `--output-schema`; rule-based fallback |
| R11 | Distribution | Signed + notarized `.app` via GitHub Releases in v1; signing keys/certificates are configured manually by the maintainer |

### 1.4 Golden path (v1 acceptance scenario)

The v1 product is complete when this scenario works end-to-end with no keyboard
or mouse after step 1 (full scripts: Appendix A):

1. Press the global hotkey. Earcon plays; HUD appears; mic turns on.
2. Say (ja or en): *"handsfree プロジェクトで README のタイポを直して"*.
3. Handsfree resolves the project, echoes the task, and asks for dispatch
   confirmation. Say *"はい"*. A `codex exec` turn starts (Tier 1 sandbox).
4. Progress milestones are narrated while it runs; saying *"ストップ"* would
   cancel.
5. The agent finishes with a structured result; the voice summary is spoken.
6. Say a follow-up: *"じゃあ push して"*. The agent hits the network boundary,
   returns `blocked`; Handsfree announces the exact action and a nonce phrase;
   echo the phrase; the turn resumes with escalated sandbox and pushes.
7. Completion is announced. Say *"終了"*. Mic turns off; session ends.
8. Start a long task, end the session while it runs; on completion, a
   notification + spoken announcement fires; press the hotkey and the completed
   task is offered for follow-up.

---

## 2. Scope

### 2.1 v1 scope (must all be delivered by the issue plan)

- Menu bar resident app; global hotkey session start/stop; session HUD.
- Continuous listening with VAD endpointing inside a session; half-duplex audio
  (mic gated while TTS speaks).
- On-device STT (`SpeechTranscriber`, ja-JP / en-US) and TTS
  (`AVSpeechSynthesizer`) behind provider protocols, with model/voice asset
  onboarding.
- Dialogue orchestrator with the session FSM of §5.1, intent phrase matching
  (ja/en), narration policy, nonce approval engine.
- Codex adapter: spawn/stream/cancel/resume `codex exec --json`, structured
  final output contract, risk-tier sandbox mapping (§6.5).
- Task manager: one foreground conversation, background continuation, numbered
  task addressing for blocked/completed tasks.
- Project registry with voice name resolution.
- Config store, transcript store with retention, structured logging with
  redaction.
- Settings window, onboarding (permissions, speech assets, codex preflight),
  notifications.
- Security hardening per §9; signed + notarized release pipeline; CI.
- User docs: README, PRIVACY, SECURITY, CONTRIBUTING, manual QA checklist.

### 2.2 v1 non-goals (explicitly out)

- No wake word; no always-on microphone. Sessions start via hotkey/menu only.
- No cloud STT/TTS/LLM calls by the app itself; no API key storage.
- No barge-in (interrupting TTS by voice); mic is gated during playback (ADR-008).
- No voice switching between running tasks by name; numbered selection of
  *blocked/completed* tasks only (R8).
- No agent adapters besides Codex CLI (the protocol exists; Claude Code etc. are v2).
- No iOS/remote companion, no window/screen control, no dictation-into-editor mode.
- No auto-update mechanism (Sparkle etc.); manual release install in v1.
- No telemetry/analytics of any kind.
- App-executed release verbs (`git push` / `gh pr create` run by the app) — the
  agent performs these under Tier 2/3 approval instead (ADR-002 consequence).

### 2.3 v2 candidates (deferred, recorded for roadmap only)

Wake word session start; barge-in with echo cancellation; named multi-task voice
switching & status queries; additional agent adapters (Claude Code, opencode);
cloud STT/TTS opt-in providers; `SFSpeechRecognizer` provider for macOS 15;
pre-granted tier elevation per task; app-executed git verbs; Sparkle auto-update;
Homebrew cask; localization of UI strings beyond en/ja.

---

## 3. System architecture

### 3.1 Module map

Pure SwiftPM package (ADR-006), four production targets + test targets:

```
┌────────────────────────────────────────────────────────────┐
│ HandsfreeApp (executable)                                  │
│  AppKit/SwiftUI shell: menu bar, HUD, settings, onboarding │
│  notifications, hotkey registration                        │
└──────────────┬─────────────────────────────────────────────┘
               │ depends on
┌──────────────▼─────────────────────────────────────────────┐
│ HandsfreeCore (library)                                    │
│  SessionFSM · DialogueOrchestrator · IntentMatcher         │
│  ApprovalEngine · NarrationPolicy · TaskManager            │
│  ProjectRegistry · ConfigStore · TranscriptStore · Logging │
└─────┬──────────────────────────────────────┬───────────────┘
      │ uses protocols                       │ uses protocols
┌─────▼───────────────────────┐  ┌───────────▼───────────────┐
│ HandsfreeSpeech (library)   │  │ HandsfreeAgent (library)  │
│  AudioEngineManager         │  │  AgentAdapter protocol    │
│  STTProvider protocol       │  │  AgentEvent model         │
│   └ AppleSTTProvider        │  │  CodexExecAdapter         │
│  VADProvider protocol       │  │  ProcessRunner            │
│   └ SpeechDetectorVAD       │  │  JSONLDecoder             │
│  TTSProvider protocol       │  │  ResponseContract         │
│   └ AppleTTSProvider        │  │  CodexPreflight           │
│  SpeechOutputArbiter        │  │  SandboxTierMapper        │
│  Earcons                    │  │  FakeCodex (test fixture) │
└─────────────────────────────┘  └───────────────────────────┘
```

Dependency rules (enforced by SwiftPM target graph):

- `HandsfreeCore` depends on **no Apple UI/audio frameworks** — it consumes
  `STTProvider`/`TTSProvider`/`AgentAdapter` protocols. Protocols are declared
  in the module that owns the domain (`HandsfreeSpeech`, `HandsfreeAgent`);
  Core imports both libraries but only their protocol/data surfaces.
- Only `HandsfreeApp` imports AppKit/SwiftUI/UserNotifications.
- Only `HandsfreeSpeech` imports Speech/AVFoundation.
- Only `HandsfreeAgent` touches `Foundation.Process`.
- **Zero third-party runtime dependencies** in all production targets (ADR-010).

### 3.2 Process model

- Single app process (`Handsfree.app`, `LSUIElement=true` — no Dock icon).
- Each agent turn spawns one child process:
  `codex exec … --json` or `codex exec resume <thread_id> … --json`.
  Child stdout is the JSONL event stream; stdin is closed; stderr captured to
  the task log. Process group is used so cancellation kills descendants.
- Long-running turns survive session end (background continuation): the child
  belongs to the app process, not to the voice session.
- App relaunch with orphaned tasks: transcript store records `thread_id` and
  spawn PID; on startup, unknown-state tasks are marked `interrupted` and
  offered for resume (§6.6). Handsfree never re-attaches to a still-running
  orphan process (it kills by PID+start-time match if present, else marks lost).

### 3.3 Repository layout

```
Package.swift
Sources/
  HandsfreeApp/            # @main, MenuBarController, HUDWindow, SettingsWindow,
                           # OnboardingWindow, HotkeyManager, NotificationManager
  HandsfreeCore/           # SessionFSM/, Dialogue/, Approval/, Tasks/,
                           # Projects/, Config/, Transcripts/, Logging/, Phrases/
  HandsfreeSpeech/         # Audio/, STT/, VAD/, TTS/, Arbiter/, Earcons/
  HandsfreeAgent/          # Adapter/, Codex/, Contract/, Preflight/
  HandsfreeTestSupport/    # test-only library: TestClock, mocks (never linked into the app)
  FakeCodex/               # scripted codex stand-in executable (test/demo)
Tests/
  HandsfreeCoreTests/  HandsfreeSpeechTests/  HandsfreeAgentTests/
  Fixtures/                # recorded JSONL streams, FakeCodex scripts, phrase cases
Resources/                 # non-SwiftPM build inputs (consumed by scripts)
  Info.plist.template  Handsfree.entitlements  icon/
# SwiftPM target resources live inside their owning target:
#   Sources/HandsfreeCore/Resources/phrases/{ja,en}.json, scaffolds/*.txt
#   Sources/HandsfreeAgent/Resources/response-schema.json
#   Sources/HandsfreeSpeech/Resources/earcons/*.caf
scripts/
  make-app.sh  sign.sh  notarize.sh  release.sh  fake-codex/
.github/workflows/ci.yml  release.yml
docs/  (this tree)
Makefile
```

---

## 4. Speech subsystem (`HandsfreeSpeech`)

### 4.1 Audio capture lifecycle

- `AudioEngineManager` owns one `AVAudioEngine`. Mic tap is installed **only**
  between session start and session end (R6). Outside sessions the engine is
  stopped and no audio object holds the input device.
- Session start: engine prewarm target ≤ 500 ms hotkey→listening (§15).
- Device changes (default input switch, AirPods connect) mid-session: engine is
  rebuilt, a short "device changed" earcon plays, listening resumes; the
  current utterance buffer is discarded.
- Audio format: input node native format → converted as required by the STT
  provider (SpeechTranscriber consumes `AnalyzerInput` buffers).
- Raw audio is **never written to disk** (§9.4).

### 4.2 STT provider

```swift
public protocol STTProvider: Sendable {
    var id: STTProviderID { get }                 // "apple"
    func availability(locale: Locale) async -> STTAvailability
    // .available | .assetDownloadRequired(size) | .unsupportedLocale | .unauthorized
    func prepare(locale: Locale) async throws     // triggers asset download if needed
    func startStream(locale: Locale, audio: AsyncStream<CapturedBuffer>)
        -> AsyncThrowingStream<STTResult, Error>
    // CapturedBuffer is the capture type defined in §4.1; conversion to the
    // Speech framework's AnalyzerInput happens inside the Apple provider.
    func stopStream() async
}

public struct STTResult: Sendable {
    public let text: String
    public let isFinal: Bool          // volatile vs finalized
    public let confidence: Double?    // nil if provider doesn't expose it
    public let audioRange: ClosedRange<TimeInterval>?
}
```

- v1 implementation: `AppleSTTProvider` wrapping `SpeechAnalyzer` +
  `SpeechTranscriber` (macOS 26+, on-device; ja-JP/en-* confirmed — research
  doc 2026-07-08). Locale from config `general.locale_mode`; `auto` uses the
  session-start locale setting, not per-utterance detection (v1 keeps one
  active STT locale per session; switching locale = config/Settings action).
- Asset management: availability check surfaces `assetDownloadRequired`;
  onboarding and Settings drive `prepare()` with progress UI.
- Authorization: microphone TCC always; the issue must empirically pin whether
  the SpeechAnalyzer path also triggers speech-recognition TCC and handle both.

### 4.3 VAD and endpointing

- `SpeechDetectorVAD` wraps the Speech framework's `SpeechDetector` module,
  composed into the same `SpeechAnalyzer` pipeline.
- Endpointing policy (utterance finalization) — an utterance is considered
  complete when **either**:
  1. VAD reports speech end and ≥ `endpoint_silence_ms` (default 900 ms) of
     silence follows, **and** the transcriber has produced a finalized result; or
  2. a hard cap of 60 s of continuous speech is hit (finalize + notify user).
- Fallback strategy (pinned as a validation step): if `SpeechDetector` proves
  unreliable, endpoint on transcriber finalized-result timing plus an RMS
  silence gate computed from the tap buffers. The `VADProvider` protocol keeps
  this swappable without touching the orchestrator.
- While TTS is speaking (half-duplex, ADR-008) the STT stream is **paused**
  (buffers dropped, not queued) so the app never transcribes its own voice.

### 4.4 TTS provider and output arbiter

```swift
public protocol TTSProvider: Sendable {
    func speak(_ utterance: SpokenUtterance) -> AsyncStream<TTSEvent> // .started, .finished, .cancelled
    func stop()
    func voices(for locale: Locale) -> [VoiceDescriptor]  // id, name, quality, language
}
public struct SpokenUtterance: Sendable {
    public let text: String          // already sanitized (§9.5) and truncated
    public let localeHint: Locale
    public let priority: SpeechPriority  // .approval > .result > .narration
}
```

- v1: `AppleTTSProvider` on `AVSpeechSynthesizer`. Configured voice identifiers
  per language (`voice.tts_voice_ja` / `voice.tts_voice_en`) with fallback to
  `AVSpeechSynthesisVoice(language:)` when missing. Voice picker labels quality
  (compact/enhanced/premium) and links to System Settings for downloads
  (research: only compact voices preinstalled).
- `SpeechOutputArbiter` is the half-duplex gate: it owns a priority queue of
  utterances, pauses STT before playback, resumes after; `.approval` priority
  flushes `.narration` items; narration items older than 10 s are dropped
  unspoken (stale progress is noise).

### 4.5 Earcons

Short `.caf` cues, distinct per meaning, bundled in Resources: session start,
session end, listening-resumed, dispatch, approval-request (distinct &
mandatory, §9.2 T3), success, failure, device-changed. Volume follows system
output; no flashing UI dependence (also aids eyes-free use).

---

## 5. Dialogue subsystem (`HandsfreeCore`)

### 5.1 Session FSM

States (one active session max; `idle` means no session):

| State | Mic | Description |
|---|---|---|
| `idle` | off | No session. Menu bar shows idle icon |
| `starting` | warming | Engine prewarm, context announcement |
| `listening` | on | Streaming STT, awaiting utterance |
| `interpreting` | paused | Utterance finalized → intent match |
| `confirming_dispatch` | on | Spoken echo of task + yes/no |
| `dispatching` | paused | Spawning agent turn |
| `agent_running.narrating` | paused | TTS narrating a milestone |
| `agent_running.listening_limited` | on | Only stop/cancel/status intents active |
| `awaiting_input` | on | Agent asked a question (status `needs_input`) |
| `awaiting_approval.announce` | paused | Speaking action + nonce phrase |
| `awaiting_approval.await_echo` | on | Only echo/deny/cancel intents active |
| `awaiting_approval.await_screen_confirm` | on | Tier 3: HUD button must be clicked |
| `speaking_result` | paused | Voice summary playback |
| `ending` | off | Goodbye earcon, teardown |
| `error_recoverable` | on | Spoken error, returns to `listening` |

Transition table (triggers → target; unspecified triggers are ignored in that
state):

| From | Trigger | To |
|---|---|---|
| idle | hotkey / menu "Start" / "Continue task N" | starting |
| starting | engine ready | listening (announce context: bound task or default project) |
| listening | final utterance | interpreting |
| listening | idle timeout `general.idle_timeout_sec` (default 30 s) | speaking "続けますか?" → 15 s more → ending |
| listening | hotkey | ending |
| interpreting | intent `new_task` | confirming_dispatch |
| interpreting | intent `follow_up` (bound task exists) | dispatching (resume) |
| interpreting | intent `answer` (in awaiting_input context) | dispatching (resume) |
| interpreting | intent `switch_project` | listening (project set, announce) |
| interpreting | intent `end_session` | ending |
| interpreting | intent `task_select(N)` (selection offered) | listening (bind task N, announce its state) |
| interpreting | no intent match | error_recoverable ("聞き取れませんでした") |
| confirming_dispatch | yes | dispatching |
| confirming_dispatch | no / cancel | listening (task discarded) |
| dispatching | spawn ok | agent_running.listening_limited |
| dispatching | spawn error | error_recoverable |
| agent_running.* | narration due | agent_running.narrating → back to listening_limited |
| agent_running.* | intent `cancel_current` | (SIGINT process group) → speaking_result (cancelled) |
| agent_running.* | turn result `ok`/`failed` | speaking_result |
| agent_running.* | turn result `needs_input` | awaiting_input (speak question) |
| agent_running.* | turn result `blocked` | awaiting_approval.announce |
| awaiting_input | final utterance | interpreting (as `answer`) |
| awaiting_approval.announce | TTS done | await_echo |
| awaiting_approval.await_echo | exact nonce echo | Tier 2: dispatching (escalated resume) · Tier 3: await_screen_confirm |
| awaiting_approval.await_echo | deny / cancel / 20 s timeout / 2 failed echoes | speaking_result (denied; task remains resumable) |
| awaiting_approval.await_screen_confirm | HUD Approve clicked | dispatching (escalated resume) |
| awaiting_approval.await_screen_confirm | HUD Deny / 60 s timeout | speaking_result (denied) |
| speaking_result | TTS done | listening (follow-up window) |
| ending | teardown done | idle |
| any | fatal audio/agent error | speaking error → ending |

Session-end with running task: allowed from any `agent_running` state via
`end_session` ("バックグラウンドで続けます") — the task detaches to the
TaskManager (§6.6) and the FSM proceeds to `ending`.

Approval-boundary note: the `awaiting_approval.*` sub-stages mirror the
ApprovalEngine's progress. Nonce generation, echo verification, retry counting,
and the approval-internal timeouts (20 s echo / 60 s screen confirm) live in
the ApprovalEngine (§5.4), which emits stage-change and decision events; the
session FSM only transitions on those events. The HUD's Approve/Deny clicks
are routed to the ApprovalEngine, not reduced by the session FSM.

### 5.2 Intents and phrase matching

Intent set (v1, closed):

| Intent | Active in | Example ja / en |
|---|---|---|
| `new_task(text)` | listening | 「〜して」 free text (default when no other intent matches and no bound context expects an answer) |
| `follow_up(text)` | listening (after result, bound task) | 「じゃあテストも直して」 |
| `answer(text)` | awaiting_input | free text |
| `yes` / `no` | confirming_dispatch | 「はい/いいえ」 "yes/no" |
| `approve_echo(d1,d2)` | await_echo | 「承認 4 9」 "confirm 4 9" |
| `deny` | await_echo, confirming | 「拒否」「やめて」 "deny" |
| `cancel_current` | agent_running, confirming | 「ストップ」「キャンセル」 "stop" "cancel" |
| `end_session` | listening states | 「終了」「おしまい」 "end session" "goodbye" |
| `switch_project(name)` | listening | 「プロジェクト◯◯」 "project ◯◯" |
| `task_select(N)` | listening (selection offered) | 「1番」 "task one" / "number one" |
| `repeat_last` | listening | 「もう一回」 "repeat" |
| `status_query` | listening_limited, listening | 「どうなってる?」 "status" |
| `help` | listening | 「ヘルプ」 "help" |

- Matching pipeline: NFKC normalize → lowercase → strip punctuation/fillers →
  exact/alias table lookup (per-locale JSON dictionaries,
  `Resources/phrases/{ja,en}.json`, schema in Appendix B) → for command intents
  a bounded edit-distance (≤1 per 4 chars) fuzzy pass. Free text falls through
  to `new_task`/`follow_up`/`answer` by state context.
- **Nonce digits are matched exactly, no fuzzing** (§9.2 T1/T2). Digit
  vocabulary avoids ja homophones (7 = 「なな」, never 「しち」).
- Dictionaries are data, not code: adding a locale or synonym must not require
  logic changes (R9).

### 5.3 Narration policy

Input: `AgentEvent` stream (§6.1). Output: throttled `SpokenUtterance`s
(`.narration` priority) + HUD lines (HUD gets everything, voice gets a subset).

| Verbosity (`voice.narration_verbosity`) | Spoken |
|---|---|
| `quiet` | dispatch ack, final result, approvals, errors only |
| `milestones` (default) | + first `command_execution` per turn, then at most one progress line per 20 s (latest wins), file-change count when first exceeding 0 |
| `verbose` | + every command start, web searches, todo updates (still ≥ 5 s apart) |

Templates per item type live in the phrase dictionaries (localized). Command
lines are truncated to 60 chars at word boundaries; file paths are spoken as
`basename` only.

### 5.4 Approval engine (nonce echo)

- On a Tier ≥ 2 request the engine generates **two independent digits (0–9)**
  from `SystemRandomNumberGenerator`; announcement template (fixed, from the
  *policy layer*, never from agent text — §9.2 T3):
  - ja: 「{action} を実行します。承認するには『承認 {d1} {d2}』と言ってください。やめる場合は『拒否』」
  - en: "About to {action}. To approve, say: confirm {d1} {d2}. Say deny to stop."
- `{action}` is built from the structured `blocked_action` field, sanitized
  (§9.5), max 120 chars.
- Echo matching: exact digit sequence required; 2 mismatches or 20 s silence →
  denied. Denial is safe: the task stays `awaiting_approval:denied` and can be
  resumed with a different instruction.
- Tier 3 additionally requires clicking the HUD "Approve" button (pointer
  click only — the HUD is a non-activating panel and never takes keyboard
  focus; deliberately breaking hands-free; configurable off via
  `policy.tier3_screen_confirm=false`, which the Settings UI labels as reducing
  protection against voice spoofing).
- Every approval/denial is written to the transcript store with tier, nonce,
  matched text, and timestamps (§7.3).

---

## 6. Agent subsystem (`HandsfreeAgent`)

### 6.1 AgentAdapter protocol and event model

```swift
public protocol AgentAdapter: Sendable {
    var id: AgentAdapterID { get }                       // "codex"
    func preflight() async -> AgentPreflight             // installed? authed? version?
    func startTurn(_ request: TurnRequest) throws -> RunningTurn
}
public struct TurnRequest: Sendable {
    public let projectPath: URL
    public let prompt: String                  // scaffolded, §6.4
    public let resumeThreadID: String?         // nil = new thread
    public let tier: RiskTier                  // .t0Read | .t1Workspace | .t2Network | .t3Full
    public let model: String?
    public let timeout: Duration               // config agent.max_turn_seconds
}
public final class RunningTurn: Sendable {
    public let events: AsyncThrowingStream<AgentEvent, Error>
    public let processIdentity: ProcessIdentity?  // pid, pgid, start time — for crash recovery (§6.6)
    public func cancel() async                 // SIGINT group, escalate SIGKILL after 5 s
}
public enum AgentEvent: Sendable {
    case threadStarted(id: String)
    case turnStarted
    case item(AgentItem)                       // commandExecution, fileChange, message,
                                               // webSearch, todo, error, unknown(type:String)
    case turnEnded(TurnOutcome)                // single terminal event, exactly once
}
public struct TurnOutcome: Sendable {
    public let status: TurnStatus              // .completed | .failed(reason) | .cancelled | .timedOut
    public let contract: AgentResponse?        // parsed per §6.3 (only when .completed)
    public let rawFinalText: String?
    public let usage: TurnUsage?
    public let threadID: String?
}
```

Wire mapping: codex `turn.completed` → `.turnEnded(status: .completed, …)`;
codex `turn.failed` / stream-level error → `.turnEnded(status: .failed(…))`;
cancellation and timeout produce `.cancelled` / `.timedOut`. An
`item.completed` whose item type is `error` is an `AgentItem` and is NOT
terminal (observed codex quirk — research doc).

Unknown event/item types decode to `.unknown` and are logged, never fatal
(codex releases frequently; research doc pins this requirement).

### 6.2 CodexExecAdapter

- Spawn (new): `codex exec --json -C <project> -s <sandbox> [-c sandbox_workspace_write.network_access=true] [-m <model>] --output-schema <bundled schema> -o <tmp last-message file> <prompt>`
- Spawn (resume): `codex exec resume <thread_id> --json -C … (same flags) <prompt>`
- stdin: closed. stderr: captured to task log (ring buffer 64 KB). stdout:
  incremental line-buffered JSONL decoding; lines > 1 MB are dropped with a
  logged warning (defensive).
- Cancellation: SIGINT to the process group; SIGKILL after 5 s grace. The
  thread remains resumable afterwards (sessions persist under `~/.codex`).
- Timeout: `agent.max_turn_seconds` (default 1800 s) → treated as cancel +
  outcome `failed(timeout)`.
- Preflight (`CodexPreflight`): resolve binary (config `agent.codex_path` if
  set, else `PATH` lookup, pinned after first resolve — §9.2 T6), check
  `codex --version` against tested range, check auth state, surface actionable
  onboarding errors.
- The adapter never passes `--dangerously-bypass-approvals-and-sandbox`,
  `--ephemeral` (tasks must be resumable), or `--skip-git-repo-check`
  (projects must be git repositories — preflight validates and the registry
  refuses non-repo paths).

### 6.3 Voice output contract (response schema)

`Resources/response-schema.json` (full text: Appendix C) is passed via
`--output-schema` on every turn. Shape:

```json
{
  "status": "ok | needs_input | blocked | failed",
  "voice_summary": "1–3 plain sentences, same language as the user's utterance",
  "question": "string|null            — required when status=needs_input",
  "blocked_reason": "needs_network | needs_full_access | needs_out_of_workspace | null",
  "blocked_action": "string|null      — imperative description of the exact pending action",
  "detail": "string|null              — markdown for the HUD, never spoken",
  "proposed_next_action": "string|null — short suggestion, may be spoken after summary"
}
```

Parsing chain (each step falls through on failure, all failures logged):

1. Parse final `agent_message` item text as JSON → validate required fields.
2. Parse the `-o` last-message file the same way.
3. Fallback: `status=ok(heuristic)`, `voice_summary` = rule-based reduction of
   the raw final message (strip code blocks/links/markdown → first 2 sentences
   ≤ 280 chars) prefixed by a spoken notice that the summary is auto-generated.

`blocked` semantics: the agent could not perform an action due to sandbox
limits and stopped at a safe point. `blocked_reason` selects the escalation
target tier (§6.5); `blocked_action` feeds the approval announcement (§5.4).
Caps: the contract stores `blocked_action` up to 300 chars; the approval
announcement additionally truncates it to 120 chars (§5.4).

### 6.4 Prompt scaffold

`PromptScaffoldBuilder` wraps every user utterance:

```
<handsfree_contract>
You are driven by voice. The user cannot easily read long text.
- Perform the requested work autonomously within your sandbox.
- If sandbox limits block a required action (network, files outside the
  workspace), stop at a safe point and report status="blocked" with
  blocked_reason and a precise blocked_action.
- If you need a decision from the user, report status="needs_input" with one
  concise question.
- Your final message MUST conform to the provided output schema.
- voice_summary: 1–3 short sentences, same language as the user's request,
  no code, no URLs, no file paths unless essential.
</handsfree_contract>
<user_request locale="ja-JP">
{utterance text}
</user_request>
```

Resume turns (answers, follow-ups, approved escalations) use variants:
`<user_answer>`, `<user_follow_up>`, and for escalations a fixed
`<approved_action>` block naming ONLY the approved `blocked_action` and
instructing the agent to perform that action and nothing further at elevated
access. The scaffold templates are versioned resources, not inline strings.

### 6.5 Risk tiers ↔ sandbox mapping

| Tier | Meaning | codex flags | Approval required |
|---|---|---|---|
| T0 | read/plan only | `-s read-only` | none (session presence suffices) |
| T1 | edit workspace | `-s workspace-write` | spoken yes at dispatch (confirming_dispatch) |
| T2 | + network egress | `-s workspace-write -c sandbox_workspace_write.network_access=true` | nonce echo |
| T3 | full access | `-s danger-full-access` | nonce echo + screen confirm (default on) |

- New tasks always start at **T1** (T0 used for `status_query`-style informational
  turns). There is **no pre-granting** of T2/T3 at dispatch in v1.
- Escalation is **reactive and single-turn**: only in response to `blocked`,
  only via resume, only with the `<approved_action>` scaffold. The next turn
  drops back to T1.
- `blocked_reason` → target tier: `needs_network` → T2;
  `needs_full_access` / `needs_out_of_workspace` → T3.
- `policy.allow_tier3=false` disables T3 entirely (announced as permanently
  denied with guidance to run the action manually).

### 6.6 Task manager

- `TaskRecord`: `{ id (monotonic small int per day), project_id, thread_id,
  state, tier_history, created_at, updated_at, last_voice_summary,
  pending_question | pending_approval, turns[] }`.
  States: `running`, `awaiting_input`, `awaiting_approval`, `succeeded`,
  `failed`, `cancelled`, `denied`, `interrupted` (app crash recovery).
- Exactly one task may be **bound** to the voice session (the conversation
  target). Ending the session unbinds; the task keeps running.
- Terminal/blocked transitions of *unbound* tasks fire: user notification +
  spoken one-liner (only if no TTS is active; else queued behind current
  playback) — e.g., 「タスク2、完了。プッシュの承認待ちです」.
- Hotkey with pending tasks (`awaiting_input`, `awaiting_approval`, or
  unacknowledged terminal): session starts with a queue announcement:
  “タスク2が質問しています。続けますか?” → `yes` binds it; `task_select(N)`
  chooses another; `new_task` starts fresh. Single pending task binds on `yes`;
  multiple are enumerated (max 3 spoken, rest "ほか N 件" + HUD list).
- Cap: `max_concurrent_tasks` default 3 (spawning beyond it is refused with
  spoken guidance) — codex turns are expensive; this is a safety valve, not a
  scheduler.

---

## 7. Projects, configuration, persistence

### 7.1 Project registry

- Entry: `{ id: UUID, name: String, aliases: [String], path: URL,
  default_model: String? }`. Name + aliases feed voice resolution
  (normalized exact → fuzzy ≤ edit distance 2 → spoken disambiguation of top 2).
- Validation at registration and at each dispatch: path exists, is a directory,
  is a **git repository**, is not inside Handsfree's own support directory.
  Failures produce spoken + HUD errors, never silent fallback to another path.
- The registry is part of the config file; CRUD via Settings only (no voice
  CRUD in v1). A `default_project` id selects the session-start context.

### 7.2 Config store

- Location: `~/Library/Application Support/Handsfree/config.json`
  (`0600`, parent dir `0700`). Atomic replace writes (temp file + rename).
  Codable schema version field (`version: 1`) with forward-compatible decoding
  (unknown keys preserved on rewrite via `JSONSerialization` merge, so config
  written by a newer build isn't destroyed by an older one).
- Full key table in Appendix C. Notable defaults: locale `auto`, hotkey
  `⌃⌥Space`, verbosity `milestones`, `tier3_screen_confirm=true`,
  `transcript_retention_days=30`, `max_turn_seconds=1800`.
- No secrets exist in config by design (R3/R11); the schema comment forbids
  adding key material in future changes without a security review (§9.4).

### 7.3 Transcript store

- Per-session JSONL under
  `~/Library/Application Support/Handsfree/transcripts/YYYY/MM/<session-id>.jsonl`
  (`0600`/`0700`). Record types: `session_meta`, `utterance` (final text only —
  volatile partials are never persisted), `intent`, `dispatch`, `agent_item`
  (digested: type + truncated payload ≤ 512 chars), `approval`
  (tier, nonce, matched_text, decision), `result`, `error`.
- Raw audio is never persisted (§4.1). Task index `tasks.json` maintained
  alongside for §6.6 recovery.
- Retention: daily sweep deletes files older than `privacy.transcript_retention_days`
  (0 = keep nothing: session files are written to a temp dir and unlinked at
  session end; `store_transcripts=false` behaves the same). HUD exposes
  "Reveal transcripts in Finder" and "Delete all transcripts now".

---

## 8. UI surfaces (`HandsfreeApp`)

### 8.1 Menu bar item

- SF Symbol reflecting FSM state: idle `mic.slash`, listening `waveform`
  (animated), agent running `gearshape` (spinning badge), awaiting approval
  `exclamationmark.triangle.fill` (tinted), speaking `speaker.wave.2`.
- Menu: Start/End Session (hotkey shown), active + pending tasks (state, last
  summary; click → bind flow §6.6), project quick-pick, Settings…, Onboarding…,
  Quit. Quit with running tasks warns (tasks would be killed; offers "keep
  running until finished then quit" = graceful drain).

### 8.2 Session HUD

- Small floating non-activating panel (top-right default, draggable,
  `NSPanel .nonactivatingPanel` so keyboard focus is never stolen).
  Shows: state chip, live partial transcription, last narration lines (3),
  bound task + project, and during approvals the **full action text + Approve /
  Deny buttons** (Tier 3). HUD is informative; every critical interaction has a
  voice path except the deliberate Tier-3 click.

### 8.3 Settings window

Tabs: **General** (hotkey recorder, language mode, idle timeout, launch at
login via `SMAppService`), **Voice** (STT locale/status + asset download, TTS
voice pickers with quality labels + preview button, rate, verbosity),
**Projects** (registry CRUD table, default project), **Policy** (tier toggles
with plain-language risk explanations), **Privacy** (retention, delete-all,
reveal folder), **Agent** (codex path override, detected version/auth status,
model override, max turn seconds), **About**.

### 8.4 Onboarding (first launch, re-runnable)

1. Welcome + privacy statement (all speech processing on-device; what leaves
   the machine: codex's own traffic under your codex account, plus
   user-initiated Apple speech-model downloads during setup).
2. Microphone permission request (+ speech recognition if the API demands it).
3. STT model check/download for chosen language(s).
4. TTS voice quality: detect compact-only state, deep-link to System Settings
   voice downloads, offer preview.
5. Codex preflight: binary found? version? logged in? (guidance, no key entry).
6. Register first project (folder picker; git validation).
7. Hotkey teach + 30-second scripted first session against FakeCodex demo mode
   (safe, offline, no real agent) so the user learns the loop risk-free.

### 8.5 Notifications

`UNUserNotificationCenter` for background task transitions (§6.6). Actions:
"Start voice session" (binds task), "Dismiss". Notification text uses the same
sanitizer as TTS (§9.5); `detail` content never appears in notifications.
Content policy (T5): completed-task notifications may carry the sanitized
`voice_summary` (≤120 chars); needs-input and awaiting-approval notifications
carry only task number, project name, and state — the question/action text is
disclosed only in-session (speech/HUD) after the user re-engages, never on the
lock screen.

---

## 9. Security model

### 9.1 Trust boundaries

```
[User's voice] ──(mic, TCC)──▶ [STT on-device] ──▶ [Intent/Approval engines]
                                                        │ policy-gated
[Agent (codex) child process] ◀──(argv, sandbox tier)───┘
   │  codex's own network/auth (user's codex account, outside our TCB)
   ▼
[Project working tree (git repo)]      [codex sandbox = enforcement boundary]
[Handsfree config/transcripts]  ── local files, 0600/0700, no secrets
```

- In v1 the app's TCB includes: phrase matcher, approval engine, tier mapper,
  process spawner, config/transcript stores. The *enforcement* boundary for
  agent actions is codex's sandbox; Handsfree's job is to never request a
  weaker sandbox than the approved tier.
- Physical/local attacker with keyboard access is **out of scope** (equivalent
  to owning the account); voice-only attackers are IN scope (T2 below).

### 9.2 Threat model and mitigations

| ID | Threat | Mitigations (design section) |
|---|---|---|
| T1 | Misrecognition dispatches or approves unintended work | dispatch echo + yes gate (§5.1); nonce digits exact-match, no fuzzing (§5.2); denial-safe defaults; T1 sandbox cap for new tasks (§6.5) |
| T2 | Third-party voice / played-back audio (TV, meeting) triggers actions | sessions only via local hotkey (R6); random 2-digit nonce per approval, unpredictable to an outsider (§5.4); Tier-3 screen confirm (physical presence proof); approval timeout + retry cap |
| T3 | Prompt injection via agent output: `voice_summary` tries to social-engineer an approval or fake system speech | approval announcements built ONLY from policy templates + sanitized `blocked_action` (≤120 chars); mandatory approval earcon precedes them; TTS sanitizer strips control chars, markup/SSML-like tags, and URLs (§9.5); summaries are capped at 400 chars. Note: keyword-based "imperative content" filtering is deliberately NOT relied on — the effective defenses are the unpredictable nonce, template-only announcements, and the earcon |
| T4 | Malicious project repo content attacks the app | app never executes/parses repo content itself (no shell, no eval); only codex touches the tree inside its sandbox; paths validated (§7.1) |
| T5 | Transcript/notification leakage of sensitive code or secrets | transcripts 0600, retention sweep, opt-out (§7.3); notifications carry summaries only (§8.5); raw audio never stored (§4.1) |
| T6 | `codex_path` / binary hijack, PATH confusion | absolute-path pin after first resolution, stored in config; warn + require Settings confirmation when the resolved binary path changes; refuse world-writable binary paths (§6.2) |
| T7 | Escalated (T2/T3) turn does more than the approved action | single-turn escalation, narrow `<approved_action>` scaffold (§6.4/6.5); full narration + transcript of escalated turns; next turn auto-drops to T1 |
| T8 | Supply chain (build/CI) | zero third-party runtime deps (ADR-010); GitHub Actions pinned by commit SHA; release builds from clean checkout; checksums published with releases |
| T9 | Tampered distribution artifact | Developer ID signing + notarization + stapling (R11); SECURITY.md documents verification; no auto-update channel in v1 |
| T10 | App overreach (mic when not expected) | mic strictly bound to session FSM states (§5.1 table); menu bar + system indicators reflect it; no accessibility permission requested at all (hotkey via `RegisterEventHotKey`, which needs none) |

Abuse cases tracked as acceptance criteria in security issues: replayed
approval recording (defeated by nonce), narration loop reading agent-crafted
"confirm 4 9" (defeated by template-only announcements + earcon + the fact the
nonce isn't in agent-visible context), config edited to point codex at a rogue
binary (defeated by T6 flow).

### 9.3 Permission and entitlement model

| Permission | Why | When requested |
|---|---|---|
| Microphone (TCC) | STT input | onboarding step 2 |
| Speech recognition (TCC) | possibly required by SpeechAnalyzer path | same step, only if API mandates |
| Notifications | background task alerts | onboarding step 7 / first background task |
| — no Accessibility, no Full Disk, no Screen Recording | not needed by design | never |

Entitlements (hardened runtime): `com.apple.security.device.audio-input` only.
**App Sandbox is intentionally OFF in v1** (ADR-010): the product's core is
spawning a developer CLI against arbitrary user project paths; the mitigation
stack is hardened runtime + notarization + codex's own sandbox + this threat
model. Revisit for MAS distribution (v2+, non-goal).

### 9.4 Secret handling rules

- The app stores **no credentials of any kind**. Codex auth belongs to the
  codex CLI (`codex login`). Cloud speech (v2) must use Keychain + per-provider
  opt-in screens; this rule is a standing constraint for future PRs.
- Config/transcripts must never contain: API keys, tokens, or env dumps. The
  spawn environment for codex is the user's default login environment minus
  Handsfree-internal variables; Handsfree adds none.
- Log redaction (§13) masks `sk-…`-like and `ghp_…`-like token patterns
  defensively before write.

### 9.5 Input validation at every boundary

| Boundary | Validation |
|---|---|
| JSONL from codex | line length cap 1 MB; strict per-event decode; unknown types → `.unknown`; item counts capped (10 000/turn) |
| Final response contract | JSON Schema validation; field length caps (`voice_summary` ≤ 400, `blocked_action` ≤ 300, `question` ≤ 300); enum whitelist for `status`/`blocked_reason` |
| Text → TTS / notifications | sanitizer: strip control chars, zero-width, markup/SSML-like tags, URLs (spoken as "リンク"), collapse whitespace; length caps |
| Config file | Codable + schema version; path fields must be absolute, exist, and be user-owned; numeric ranges clamped |
| Phrase dictionaries (bundled) | schema-validated at build time by a unit test |
| STT text | treated as untrusted free text; only ever matched against dictionaries or wrapped in prompt scaffold (never interpolated into shell) |
| Process spawning | argv arrays only, never `sh -c`; project path passed via `-C`; no user text in flags. One documented exception: a once-per-run login-environment capture executes the user's own `$SHELL` (validated absolute path) with a fixed literal command string (`-l -c 'command -v codex'` / `-l -c env`) and no interpolated input; the captured environment is in-memory only, never logged or persisted, and `HANDSFREE_*`-prefixed variables are stripped before any spawn |

### 9.6 Secure defaults summary

Mic off outside sessions · new tasks at T1 · network denied until nonce
approval · Tier 3 requires physical click · transcripts local-only with 30-day
retention · no telemetry · no auto-update · no third-party code at runtime.

---

## 10. Build, packaging, release

- `swift build/test` for all logic; `make app` runs `scripts/make-app.sh`
  (assemble `Handsfree.app`: binary, `Info.plist` from template with
  `NSMicrophoneUsageDescription`, `NSSpeechRecognitionUsageDescription`,
  `LSUIElement`, version stamping; resources; `.icns` via `iconutil`).
  `make app` ad-hoc signs the bundle by default so local TCC identity stays
  stable across rebuilds; `make app SIGN=none` (used by CI) skips signing.
  Developer ID signing is exclusively `make sign` (§ release pipeline).
- `make sign` (`codesign --options runtime` + entitlements),
  `make notarize` (`notarytool submit --wait` + `stapler staple`),
  `make release` (zip, checksums). All CLT-only — verified on the dev machine
  (research doc). Secrets (`APPLE_ID`, team id, app-specific password / API
  key) come from the environment/CI secrets and are configured manually by the
  maintainer (R11); scripts fail with actionable messages when absent.
- CI (`ci.yml`): macos-26 runner, `swift build && swift test` + phrase/schema
  validation tests + `make app` smoke (unsigned). `release.yml`: tag-triggered,
  builds, signs, notarizes, uploads artifact + checksums to a GitHub Release
  (manual publish gate; merge ≠ release).

## 11. Testing & validation strategy

| Layer | Approach |
|---|---|
| FSMs (session, approval, tasks) | exhaustive table-driven unit tests, incl. timeout paths (virtual clock) |
| Phrase matching | golden-file tests per locale; adversarial cases (homophones, fillers, partial nonce) |
| JSONL decoding | recorded fixtures (from live probes) + malformed/oversize/unknown-type cases |
| Response contract | valid/invalid/fallback chains; injection corpus for the sanitizer |
| CodexExecAdapter | integration vs **FakeCodex** (scripted executable fixture: happy path, blocked, needs_input, garbage output, hang, crash, slow-drip) |
| Orchestrator E2E | mock speech + FakeCodex driving the full FSM through the golden path and each error path — runs in CI |
| Live audio/STT/TTS | not in CI; `docs/QA_CHECKLIST.md` manual pass per release (golden path ja+en on real mic) |
| Performance | budget assertions where measurable (§15) |

Every issue in the plan carries its own Validation section; the FakeCodex
fixture and mock speech providers are deliberately early issues because most
other tests depend on them.

## 12. Failure mode catalog (must be handled, spoken, and logged)

codex missing / unauthenticated / version-unsupported → onboarding guidance ·
spawn failure · JSONL parse failure mid-turn (turn continues if stream
recovers; else failed) · schema-invalid final message (fallback chain §6.3) ·
turn timeout · STT asset missing (download flow) · mic permission denied
(Settings deep link) · TTS voice deleted (language fallback) · audio device
change (§4.1) · machine sleep during turn (child continues; on wake, stream
resumes; if process died → `interrupted`) · app quit/crash with running tasks
(§3.2 recovery) · disk full on transcript write (drop transcript, keep session
alive, warn once) · session start while previous teardown incomplete
(debounce).

## 13. Logging & diagnostics

`os.Logger` subsystem `io.github.saber5656.handsfree`, categories: `audio`,
`stt`, `tts`, `dialogue`, `approval`, `agent`, `store`, `ui`. Privacy: default
`.private` for payloads; redaction pass (§9.4) before any string interpolation
marked public. A "Copy diagnostic snapshot" Settings button exports versions,
config (secrets-free by construction), last 200 log lines, task index — for
bug reports.

## 14. Internationalization

- UI strings: classic `.lproj/Localizable.strings` (en base + ja), because
  `.xcstrings` compilation needs full Xcode (toolchain constraint, research
  doc). A unit test asserts key parity between locales.
- Spoken strings and phrase dictionaries: `Resources/phrases/{ja,en}.json`
  (Appendix B schema) — adding locales is data + QA, not code.
- Docs/README English; README carries a short Japanese section.

## 15. Performance budgets

| Metric | Budget |
|---|---|
| Hotkey → listening earcon | ≤ 500 ms |
| Utterance end → intent decision | ≤ 300 ms (matching itself ≤ 10 ms) |
| Dispatch → first narration | ≤ 3 s (bounded by codex startup) |
| Agent event → HUD line | ≤ 100 ms |
| Idle footprint | < 0.5 % CPU, < 80 MB RSS, 0 audio objects held |
| In-session footprint (no turn running) | < 15 % CPU on M-series |

## 16. Known unknowns (tracked; may spawn issues during implementation)

1. `SpeechDetector` runtime behavior (result cadence, tuning) — fallback
   endpointing designed (§4.3).
2. Whether SpeechAnalyzer requires the legacy speech-recognition TCC prompt.
3. codex exec exit-code semantics and `resume` behavior after SIGINT — pinned
   by adapter validation steps; contract treats the event stream as truth.
4. Stability of the `--output-schema` enforcement across codex versions;
   fallback chain exists (§6.3).
5. `⌃⌥Space` hotkey collisions with input-source switching setups — onboarding
   forces a test + easy rebind.
6. Notarization latency/flakiness in CI — release workflow retries and can be
   run locally.
7. ja↔en code-switched utterances (English technical terms inside Japanese
   sentences) STT accuracy — mitigation candidates recorded in the STT issue
   (custom vocabulary hooks if the API exposes them).
8. Final product name (naming research doc) — pre-release branding gate.

---

## Appendix A — Golden path scripts

### A.1 Japanese

```
[hotkey] ♪earcon
App : 「セッション開始。プロジェクトは handsfree です」
User: 「README のタイポを直して」
App : 「README のタイポ修正を handsfree で実行します。よろしいですか?」
User: 「はい」
App : ♪dispatch 「開始しました」 … 「実行中: rg --files …」
App : ♪success 「README の誤字を2箇所修正しました。ビルドへの影響はありません」
User: 「じゃあ push して」
App : 「続きを実行します」 … (agent blocked)
App : ♪approval 「ブランチ fix-readme-typos を origin に push します。
      承認するには『承認 4 9』と言ってください。やめる場合は『拒否』」
User: 「承認 4 9」
App : 「承認しました。実行します」 … ♪success 「push が完了しました」
User: 「終了」
App : 「セッションを終了します」 ♪end
```

### A.2 English

```
[hotkey] ♪earcon
App : "Session started. Project: handsfree."
User: "Fix the typos in the README."
App : "I'll run: fix README typos, in handsfree. Proceed?"
User: "Yes."
App : ♪dispatch "Started." … "Running: rg --files …"
App : ♪success "Fixed two typos in the README. No build impact."
User: "Now push it."
App : "Continuing." … (agent blocked)
App : ♪approval "About to push branch fix-readme-typos to origin.
      To approve, say: confirm 4 9. Say deny to stop."
User: "Confirm 4 9."
App : "Approved. Running." … ♪success "Push complete."
User: "End session."
App : "Ending session." ♪end
```

## Appendix B — Phrase dictionary schema (`Resources/phrases/<locale>.json`)

```json
{
  "version": 1,
  "locale": "ja-JP",
  "intents": {
    "yes":            ["はい", "うん", "おけ", "オッケー", "了解"],
    "no":             ["いいえ", "いや", "だめ"],
    "deny":           ["拒否", "やめて", "中止"],
    "cancel_current": ["ストップ", "キャンセル", "止めて"],
    "end_session":    ["終了", "おしまい", "セッション終了", "バイバイ"],
    "repeat_last":    ["もう一回", "もう一度", "リピート"],
    "status_query":   ["どうなってる", "状況は", "ステータス"],
    "help":           ["ヘルプ", "使い方"]
  },
  "prefixes": {
    "switch_project": ["プロジェクト"],
    "task_select":    ["タスク", "番"]
  },
  "digits": { "0": ["ゼロ", "れい"], "1": ["いち"], "2": ["に"], "3": ["さん"],
              "4": ["よん"], "5": ["ご"], "6": ["ろく"], "7": ["なな"],
              "8": ["はち"], "9": ["きゅう"] },
  "approve_keyword": ["承認"],
  "templates": {
    "narration.command": "実行中: {command}",
    "narration.files":   "{count}ファイル変更",
    "approval.announce": "{action} を実行します。承認するには「承認 {d1} {d2}」と言ってください。やめる場合は「拒否」",
    "...": "full set enumerated in the phrase-table issue"
  }
}
```

Matcher contract: normalization NFKC → lowercase → trim fillers
(「えっと」"um" list per locale) → exact table match → bounded fuzzy for
intents (edit distance ≤ 1 per 4 chars) → **digits & approve keyword: exact
only**. Politeness-token stripping applies only to free-text command
extraction, never before exact alias lookup (so a bare politeness word cannot
be consumed into an unintended `yes`).

## Appendix C — Machine contracts

### C.1 Agent response schema (`Resources/response-schema.json`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "additionalProperties": false,
  "required": ["status", "voice_summary"],
  "properties": {
    "status":   { "type": "string", "enum": ["ok", "needs_input", "blocked", "failed"] },
    "voice_summary": { "type": "string", "minLength": 1, "maxLength": 400 },
    "question": { "type": ["string", "null"], "maxLength": 300 },
    "blocked_reason": { "type": ["string", "null"],
      "enum": ["needs_network", "needs_full_access", "needs_out_of_workspace", null] },
    "blocked_action": { "type": ["string", "null"], "maxLength": 300 },
    "detail":   { "type": ["string", "null"], "maxLength": 20000 },
    "proposed_next_action": { "type": ["string", "null"], "maxLength": 200 }
  }
}
```

### C.2 Config schema (excerpt; full table in the config issue)

```json
{
  "version": 1,
  "general": { "locale_mode": "auto|ja|en", "hotkey": {"key":"Space","modifiers":["control","option"]},
               "idle_timeout_sec": 30, "launch_at_login": false,
               "onboarding_completed": false },
  "voice":   { "stt_provider": "apple", "tts_provider": "apple",
               "tts_voice_ja": null, "tts_voice_en": null,
               "speaking_rate": 0.5, "narration_verbosity": "quiet|milestones|verbose" },
  "agent":   { "kind": "codex", "codex_path": null,
               "codex_path_confirmed_path": null, "model": null,
               "max_turn_seconds": 1800, "max_concurrent_tasks": 3 },
  "policy":  { "allow_tier3": true, "tier3_screen_confirm": true },
  "privacy": { "store_transcripts": true, "transcript_retention_days": 30 },
  "projects": { "default_project": null, "entries": [] }
}
```
