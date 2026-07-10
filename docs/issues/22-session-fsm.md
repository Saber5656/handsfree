# Title

Session state machine: pure reducer with exhaustive transition table

## Summary

Implement `HandsfreeCore/SessionFSM`: the session lifecycle of DESIGN §5.1
as a pure reducer `reduce(state, event) -> (state, [Effect])`, with timer
effects driven by the orchestrator and approval sub-stages mirrored from the
ApprovalEngine.

## Context

The FSM is the backbone every subsystem hangs off; DESIGN §5.1 fixes states,
mic policy, and transitions. Per the amended §5.1 note, approval internals
(nonce, echo, retries, approval timeouts, screen-confirm input) live in the
ApprovalEngine — the FSM only mirrors stage changes and consumes decisions.

## Scope

- State/event/effect types + reducer + local shell types + tests. Not:
  effect execution (28), matching (19), approval internals (21).

## Detailed Requirements

1. `SessionState`: exactly the DESIGN §5.1 states; `agent_running`
   sub-state (`.narrating`/`.listeningLimited`) and `awaiting_approval`
   stage (`.announce`/`.awaitEcho`/`.awaitScreenConfirm`) as associated
   values. `var micPolicy: MicPolicy` (.off/.on/.paused/.warming) matching
   the DESIGN table — the full mapping is asserted by one test.
2. Local shell types (defined here in `HandsfreeCore/SessionFSM/Types.swift`
   with exactly the fields the reducer needs; richer versions live in their
   owning modules and are mapped by the orchestrator):
   `TaskID (Int)`, `TaskBinding (.task(TaskID) | .fresh)`,
   `TaskDigest { id, state, projectName, summaryKeyOrText }`,
   `TurnStatusDigest (.ok(voiceSummary: String, isFallback: Bool,
   proposedNext: String?) | .needsInput(question: String) |
   .blocked(action: String, targetTier: RiskTier) | .failed(reasonKey:
   String) | .cancelled | .timedOut)`,
   `TaskDraft { utterance, project }`, `TurnPlan { binding, kind, tier,
   utterance }`, `SpeakSpec (.template(key: String, args: [String: String])
   | .sanitizedText(String, locale: SpeechLocale))`,
   `TranscriptRecordDraft` (enum mirroring 27's record types).
   Imports allowed: `HandsfreeAgent` (RiskTier, AgentEvent), nothing else
   beyond Foundation.
3. `SessionEvent` (complete): `hotkeyPressed`,
   `startRequested(TaskBinding?)`, `engineReady`, `engineFailed(reasonKey)`,
   `utteranceFinal(String)`, `intentResolved(MatchResult)`,
   `dispatchSucceeded(TaskID)`, `dispatchFailed(reasonKey)`,
   `agentEvent(TaskID, AgentEvent)`, `turnOutcome(TaskID, TurnStatusDigest)`,
   `approvalStageChanged(ApprovalStage)`,
   `approvalDecision(ApprovalDecision)`, `ttsFinished(UtteranceKind)`,
   `timerFired(TimerKind)`, `endRequested`, `fatalError(reasonKey)`.
   (No `screenConfirm` event — HUD clicks route to the ApprovalEngine.)
4. `Effect` (complete): `startAudio`, `stopAudio`, `pauseSTT`, `resumeSTT`,
   `speak(SpeakSpec, priority: SpeechPriority)`, `playEarcon(Earcon)`,
   `matchIntent(text: String, context: MatchContextSpec)`,
   `confirmDispatch(TaskDraft)`, `dispatch(TurnPlan)`, `cancelTurn(TaskID)`,
   `beginApproval(taskID: TaskID, action: String, tier: RiskTier)`,
   `resumeWithApproval(TaskID, RiskTier)`, `bindTask(TaskID)`, `unbindTask`,
   `announcePendingQueue([TaskDigest])`, `startTimer(TimerKind, Duration)`,
   `cancelTimer(TimerKind)`, `persist(TranscriptRecordDraft)`, `teardown`,
   `noop(ignored: IgnoredEventNote)` — `noop` IS an enum case so tests can
   assert intentional ignores (`IgnoredEventNote { stateName, eventName }`).
5. Transitions: every row of the DESIGN §5.1 table, plus: unspecified
   (state, event) pairs → `[.noop(…)]`; idle-timeout two-stage
   (`idleListening` 30 s → continue prompt → `continuePromptWindow` 15 s →
   ending); `approvalDecision(.approved(tier))` → dispatching via
   `resumeWithApproval`; `approvalDecision(.denied(…))` → speaking_result
   (denied template; task stays resumable); session end during
   `agent_running` → effects `[unbindTask, speak(bg-continue), teardown]`.
   `TimerKind = .idleListening | .continuePromptWindow` ONLY (utterance cap
   is endpointing's, approval timeouts are the engine's — documented).
6. Sanitization invariant (T3): `SpeakSpec.sanitizedText` is the only way
   free text enters `speak`, and the reducer only ever constructs it from
   `TurnStatusDigest` fields — which the orchestrator guarantees are
   sanitized before reduction (documented contract; enforced by 28's tests).
7. Coverage tooling: `SessionEventKind` (associated-value-free mirror) +
   fixture factories for concrete events; a generated coverage matrix
   (state × event-kind) is written by a test to
   `.build/session-fsm-coverage.md` and the test FAILS if any DESIGN-table
   row is missing (staleness guard).
8. Property test: 10 000 random event sequences (seeded generator) — no
   crash, always a defined state, every ignore is an explicit `.noop`.

## Acceptance Criteria

- [ ] One test per DESIGN §5.1 table row asserting new state AND ordered
      effect list (reviewer can diff table↔tests via the coverage matrix).
- [ ] Mic-policy mapping test matches DESIGN exactly.
- [ ] Idle-timeout two-stage sequence via direct `timerFired` events.
- [ ] Approval mirroring: stage changes reflect engine events; decisions
      drive resume/denied paths; no echo/nonce logic present in the reducer
      (code review + grep for "nonce" returns nothing).
- [ ] Detach-on-end flow test.
- [ ] Property test green; coverage artifact generated and complete.
- [ ] Imports limited to Foundation + HandsfreeAgent (compile-level check
      documented).

## Validation

`swift test --filter SessionFSMTests`; coverage artifact path noted in PR.

## Dependencies

12, 19 (types only: `MatchResult`, `IntentKind`).

## Non-goals

Effect execution/wiring (28), narration content (23), approval internals
(21), timer scheduling implementation (28's TimerDriver).

## Design References

DESIGN.md §5.1 (amended approval-boundary note), §5.4, §6.6, §12; ADR-008,
ADR-009.
