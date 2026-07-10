# Title

Session state machine: pure reducer with exhaustive transition table

## Summary

Implement `HandsfreeCore/SessionFSM`: the session lifecycle of DESIGN §5.1 as
a pure, table-tested reducer `reduce(state, event) -> (state, [Effect])`, with
timeouts driven by injected clocks.

## Context

The FSM is the backbone every subsystem hangs off; DESIGN §5.1 fixes its
states, mic policy per state, and full transition table. Keeping it pure (no
I/O) is what makes the golden-path E2E (38) and this issue's exhaustive tests
possible in CI.

## Scope

- State/event/effect types + reducer + timeout scheduling model + tests.
- Not: effect execution (28), intents (19), approval internals (21 — the FSM
  treats approval as a sub-flow it delegates to and receives a decision from).

## Detailed Requirements

1. `SessionState` enum: exactly the states of DESIGN §5.1 (including
   `agent_running.narrating` / `.listening_limited` as an associated sub-state,
   and `awaiting_approval` with stage). Each state exposes
   `var micPolicy: MicPolicy (.off/.on/.paused/.warming)` matching the DESIGN
   table (unit test asserts the whole mapping).
2. `SessionEvent` enum (complete):
   `hotkeyPressed, startRequested(binding: TaskBinding?), engineReady,
   engineFailed(AudioError), utteranceFinal(String),
   intentResolved(MatchResult), dispatchSucceeded(TaskID),
   dispatchFailed(reason), agentEvent(TaskID, AgentEvent),
   turnOutcome(TaskID, TurnStatusDigest), approvalDecision(ApprovalDecision),
   ttsFinished(utteranceKind), timerFired(TimerKind), screenConfirm(Bool),
   endRequested, fatalError(reason)`.
3. `Effect` enum (complete):
   `startAudio, stopAudio, pauseSTT, resumeSTT, speak(SpeakSpec),
   playEarcon(Earcon), matchIntent(text, MatchContext), confirmDispatch(taskDraft),
   dispatch(TurnPlan), cancelTurn(TaskID), beginApproval(ApprovalRequest),
   resumeWithApproval(TaskID, RiskTier), bindTask(TaskID), unbindTask,
   announcePendingQueue([TaskDigest]), startTimer(TimerKind, Duration),
   cancelTimer(TimerKind), persist(TranscriptRecordDraft), teardown`.
4. Transition coverage: implement EVERY row of the DESIGN §5.1 table, plus:
   - unspecified (state, event) pairs are ignored with a debug log effect
     (explicit `.noop(logged:)` so tests can assert intentional ignores);
   - idle-timeout two-stage flow (30 s → continue prompt → 15 s → ending);
   - approval timeout handling arrives as `approvalDecision(.denied(.timeout))`
     (21 owns approval-internal timers; the FSM owns listening/idle timers —
     document the timer ownership split);
   - session end during `agent_running` detaches the task (effects:
     `unbindTask, speak(bg-continue), teardown`).
5. Timer model: reducer emits `startTimer/cancelTimer` effects; a
   `TimerDriver` (in 28) feeds back `timerFired`. `TimerKind`:
   `idleListening, continuePromptWindow, utteranceCap` (60 s cap belongs to
   endpointing 09 — NOT here; document).
6. Determinism: reducer is a pure function; state+event fully determine
   output. Add `func validTransitions(from:) -> [SessionEvent]` used by a
   property test that random event sequences never crash and always land in a
   defined state.

## Acceptance Criteria

- [ ] Table-driven test with one case per DESIGN §5.1 row (each asserts new
      state AND full ordered effect list) — reviewer can diff table↔tests.
- [ ] Mic-policy mapping test matches DESIGN exactly.
- [ ] Idle-timeout two-stage sequence test (TestClock via TimerDriver stub).
- [ ] Property test: 10 000 random event sequences, no crash, no undefined
      state, every `.noop` is the documented-ignore kind.
- [ ] Detach-on-end flow test (running task survives; effects exact).
- [ ] Reducer has zero imports beyond the domain modules (no Foundation
      process/audio/UI imports).

## Validation

`swift test --filter SessionFSMTests` — includes a generated coverage matrix
(state × event) dumped as a test artifact for review.

## Dependencies

12, 19 (types only: `MatchResult`, `AgentEvent`, `ApprovalDecision` shells).

## Non-goals

Effect execution/wiring (28), narration content (23), approval internals (21).

## Design References

DESIGN.md §5.1 (authoritative table), §5.4, §6.6, §12; ADR-008, ADR-009.
