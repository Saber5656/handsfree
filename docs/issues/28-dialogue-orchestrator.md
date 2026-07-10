# Title

DialogueOrchestrator: wire FSM, speech, approval, agent, tasks — mock E2E

## Summary

Implement `HandsfreeCore/Dialogue/DialogueOrchestrator`: the composition
root binding the session FSM to all subsystems, enforcing the sanitization
boundary, exposing the app-facing session API, and proving the full loop
with CI-runnable mock E2E tests.

## Context

Every component so far is pure/isolated; this is the composition root
(DESIGN §5.1/§6.x). The app shell (29), HUD (31), notifications (35), and
onboarding demo (34) all drive the public API defined here.

## Scope

- Orchestrator actor + effect executor + TimerDriver + public API + mock E2E
  suite. Not: AppKit UI, live audio.

## Detailed Requirements

1. Construction (protocol-typed DI): `init(stt:, vad:, endpointing:, tts:,
   arbiter:, matcher:, approval:, scaffold:, registry:, tasks:, transcripts:,
   adapter:, config:, clock:)`.
2. Public API (consumed by 29/31/34/35; exact):
   ```swift
   public func hotkeyPressed() async
   public func startSession(binding: TaskBinding?) async     // menu/notification entry
   public func endSession() async
   public func screenConfirm(_ approved: Bool) async         // HUD buttons → ApprovalEngine
   public var stateStream: AsyncStream<SessionStateSnapshot> // FSM state + bound task + HUD lines + approval stage + mode
   public var mode: OrchestratorMode                          // .normal | .demo (34 sets .demo)
   ```
3. Event pump: single actor loop funneling external inputs (public API,
   task events, STT results, endpointing decisions, arbiter gate, approval
   effects, agent events, timer fires) into `SessionEvent`s; FSM effects
   executed in order. One executor function per `Effect` case (a test per
   case). `TimerDriver` implements `startTimer/cancelTimer` on the injected
   clock and feeds `timerFired` back.
4. **Sanitization boundary (T3 — enforced here, tested here)**: every
   agent-derived string is passed through `SpeechTextSanitizer` at the
   moment a `TurnStatusDigest` is built from a `TurnOutcome` — voice_summary
   (≤ 400), question (≤ 300), blocked_action (≤ 120 for approval usage),
   proposed_next_action (≤ 200). Downstream (FSM `SpeakSpec`, approval
   request, HUD lines from 23, TaskManager digests) only ever sees
   sanitized values. Injection tests: hostile contract fields (SSML, URLs,
   control chars, fake approval instructions) for result/question/blocked
   paths.
5. Cross-cutting behaviors (each with a dedicated test):
   - Half-duplex: arbiter `sttGate=false` pauses STT ingestion; utterances
     finalized during the gate are DISCARDED (documented UX rule).
   - `blocked_reason`→tier: needs_network→t2; needs_full_access /
     needs_out_of_workspace→t3; `allow_tier3=false` → spoken
     `approval.tier3_disabled`, task → denied WITHOUT starting the approval
     flow.
   - Escalated resumes use scaffold kind `.approvedEscalation` with the
     approved action text and drop back to T1 on the following turn
     (invocation-log assertion via FakeCodex).
   - Fallback summary → `announce.fallback_summary_notice` prefix.
   - Session-start binding (§6.6): pending queue announced (≤ 3 spoken +
     "ほか N 件"), `yes` binds the single/first, `task_select(N)` binds N,
     `new_task` starts fresh with the default project.
   - Locale: `ja`→ja-JP, `en`→en-US, `auto`→ja-JP iff the system UI language
     is Japanese else en-US; chosen once at session start; passed to
     STT/matcher/scaffold/TTS consistently (test all three modes).
   - Transcript records at: session start/end, final utterance, intent,
     dispatch, approval (audit record from 21), result, error.
   - Narration: `NarrationPolicy.beginTurn` receives the tier/escalation
     context (T7 full-narration override verified end-to-end).
6. Mock E2E suite (`HandsfreeCoreTests/E2E/`; MockSTT/MockTTS/MockVAD +
   REAL CodexExecAdapter against FakeCodex): scripts utterances as STT
   finals and asserts the ordered spoken output (MockTTS record) + task/
   transcript end-state:
   1. Golden path ja (Appendix A.1: happy then blocked-network escalation).
   2. Golden path en (A.2).
   3. needs_input loop.  4. Approval denied (echo fail ×2) → denied,
   resumable.  5. Cancel mid-turn.  6. Idle timeout two-stage → end.
   7. Session end with running task → background completion event.
   8. Dispatch-confirm "no" discards.  9. Ambiguous project disambiguation.
   10. `garbage` scenario → fallback notice spoken.
7. Performance: a wall-clock test (`ContinuousClock`, mocked deps, no
   artificial delays) asserts utterance-final → intent-decision executor
   latency ≤ 300 ms with a 3× CI-tolerance multiplier documented (DESIGN
   §15's real budget is re-measured manually in 38/40).

## Acceptance Criteria

- [ ] One executor test per Effect case.
- [ ] All 10 E2E scenarios green in CI (no live suites).
- [ ] Sanitization-boundary injection tests green (result/question/blocked).
- [ ] Cross-cutting tests green (gate discard, tier mapping incl. disabled
      T3, escalation drop-back, locale matrix, pending-queue binding,
      transcript emission points, T7 narration override).
- [ ] No AppKit/AVFoundation imports (grep in Validation).

## Validation

`swift test --filter 'DialogueOrchestratorTests|E2ETests'`;
`grep -rn "^import" Sources/HandsfreeCore/Dialogue/DialogueOrchestrator.swift`
in PR.

## Dependencies

07, 10, 11, 17, 18, 21, 22, 23, 24, 25, 26, 27.

## Non-goals

UI rendering, real-audio integration (38/QA), wake word, barge-in.

## Design References

DESIGN.md §5.1–5.4, §6.3–6.6, §7.3, §9.2 (T3/T7), Appendix A; ADR-004,
ADR-005, ADR-008, ADR-009.
