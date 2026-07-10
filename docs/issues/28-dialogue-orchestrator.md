# Title

DialogueOrchestrator: wire FSM, speech, approval, agent, tasks — mock E2E

## Summary

Implement `HandsfreeCore/Dialogue/DialogueOrchestrator`: the effect executor
that binds the session FSM (22) to real subsystems — speech providers/arbiter,
intent matcher, approval engine, prompt scaffold, project registry, task
manager, transcript store, and the agent adapter — and prove the full loop
with CI-runnable mock E2E tests.

## Context

Every component so far is deliberately pure/isolated; this issue is the
composition root of the product (DESIGN §5.1/§6.x federation). After it, the
app shell (29) only needs to render state and forward hotkey/UI events.

## Scope

- Orchestrator actor + effect execution + timer driver + session lifecycle +
  mock E2E suite. Not: AppKit UI (29/31), live audio (waves 1 `live` tests).

## Detailed Requirements

1. Construction (dependency injection, all protocol-typed):
   `init(stt:, vad:, tts:, arbiter:, matcher:, approval:, scaffold:, registry:,
   tasks:, transcripts:, adapter:, config:, clock:)`.
2. Event pump: single-threaded actor loop — external inputs (hotkey, UI
   button, task events, STT results, arbiter gate, approval effects, agent
   events, timer fires) are funneled into `SessionEvent`s and reduced through
   the FSM; the returned `[Effect]` is executed in order. Effect→subsystem
   mapping table (one function per effect, DESIGN §5.1 effects list from 22):
   e.g. `.dispatch(TurnPlan)` → scaffold.build → registry.validateForDispatch
   → tasks.create/attach → adapter.startTurn (tier from plan) → forward events;
   `.beginApproval` → approval.begin + forward its effects into
   speak/screen-confirm events; `.speak` → sanitizer already applied upstream
   → arbiter.enqueue; etc. The full mapping must be written out in code with
   one test per effect kind.
3. Cross-cutting behaviors (each with a dedicated test):
   - Half-duplex: arbiter `sttGate=false` pauses STT ingestion (utterances
     finalized during gate are DISCARDED, not queued — half-duplex means the
     user knows to wait; document).
   - `blocked_reason`→tier mapping (DESIGN §6.5): needs_network→t2,
     needs_full_access/needs_out_of_workspace→t3; `allow_tier3=false` →
     spoken permanent denial (template key) without approval flow.
   - Fallback-summary notice: `isFallbackSummary` prefixes
     `announce.fallback_summary_notice`.
   - Session start binding flow (DESIGN §6.6): pending tasks →
     `announcePendingQueue` (≤ 3 spoken + "ほか N 件"), `yes`/`task_select`
     binding, else fresh context with default project.
   - Locale: session locale from config (`auto` → system UI language ja/en,
     else en); passed to matcher/scaffold/TTS consistently.
   - Transcript records emitted at: session start/end, every final utterance,
     intent, dispatch, approval, result, error (cross-check with 27's types).
4. Mock E2E suite (`HandsfreeCoreTests/E2E/`, CI-safe: MockSTT/MockTTS/
   MockVAD scripts + FakeCodex via real CodexExecAdapter): script utterances
   as STT finals and assert the SPOKEN output sequence (MockTTS record) plus
   task/transcript end-state:
   1. Golden path ja (DESIGN Appendix A.1 verbatim, FakeCodex `happy` then
      `blocked-network`+escalation).
   2. Golden path en (A.2).
   3. needs_input loop (question → answer → complete).
   4. Approval denied (echo fail ×2) → task `denied`, resumable.
   5. Cancel mid-turn (「ストップ」).
   6. Idle timeout two-stage → session end.
   7. Session end with running task → background completion event.
   8. Dispatch-confirm "no" discards.
   9. Ambiguous project disambiguation.
   10. `garbage` scenario → fallback summary notice spoken.
5. Performance guard: utterance-final → intent decision ≤ 300 ms budget
   asserted with the test clock pipeline (logic-only proxy for DESIGN §15).

## Acceptance Criteria

- [ ] Effect mapping: one unit test per effect kind (all pass).
- [ ] All 10 E2E scenarios green in CI (no `live` tags).
- [ ] Cross-cutting behavior tests (gate discard, tier mapping incl.
      allow_tier3=false, fallback notice, pending-queue binding, locale
      consistency, transcript emission points) green.
- [ ] No AppKit/AVFoundation imports in the orchestrator (compile check).

## Validation

`swift test --filter DialogueOrchestratorTests E2ETests` — this suite becomes
the regression backbone extended by 38.

## Dependencies

07, 10, 11, 17, 18, 21, 22, 23, 24, 25, 26, 27.

## Non-goals

UI rendering, real-audio integration (38/QA), wake word, barge-in.

## Design References

DESIGN.md §5.1–5.4, §6.3–6.6, §7.3, Appendix A; ADR-004, ADR-005, ADR-008,
ADR-009.
