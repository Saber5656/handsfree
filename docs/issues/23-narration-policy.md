# Title

NarrationPolicy: verbosity-tiered, throttled progress narration

## Summary

Implement `HandsfreeCore/Dialogue/NarrationPolicy`: convert the `AgentEvent`
stream into (a) throttled spoken narration and (b) unthrottled HUD lines,
per verbosity level — with forced full narration during escalated turns.

## Context

DESIGN §5.3 fixes the verbosity table; ADR-008 makes throttling essential
(every spoken line blocks the mic). Threat T7 requires escalated (approved
T2/T3) turns to be fully narrated regardless of the user's verbosity
setting.

## Scope

- Policy engine + template usage + tests. Not: arbiter queueing (11), event
  production (18), result/approval/error speech (22/28).

## Detailed Requirements

1. API:
   ```swift
   public struct TurnNarrationContext: Sendable {
       public let tier: RiskTier
       public let isEscalated: Bool          // approved T2/T3 resume turn
   }
   public struct NarrationPolicy {
       public init(verbosity: NarrationVerbosity, locale: SpeechLocale,
                   templates: PhraseTemplates)      // from 19
       public mutating func beginTurn(_ context: TurnNarrationContext, at now: Duration)
       public mutating func ingest(_ event: AgentEvent, at now: Duration) -> NarrationOutput
   }
   public struct NarrationOutput {
       public let spoken: SpokenUtterance?   // priority .narration
       public let hud: [HUDLine]
   }
   public struct HUDLine: Equatable, Sendable { public let icon: HUDIcon; public let text: String }
   ```
   Pure value type: time is injected via `now` (no internal clocks); the
   caller (28) supplies monotonic time.
2. **Escalation override (T7)**: when `isEscalated == true`, the effective
   verbosity is `verbose` regardless of configuration. Tests: quiet+escalated
   and milestones+escalated both narrate every command.
3. HUD mapping (exhaustive; every `AgentEvent`/`AgentItem` case):
   | Event/Item | HUD line |
   |---|---|
   | `threadStarted`, `turnStarted` | none (state chip covers it) |
   | `commandExecution(cmd, status)` | icon `terminal`, text = template `narration.command` (`.running`) or `narration.command_done` (`.succeeded`/`.failed` with status suffix) |
   | `fileChange` | icon `doc`, `narration.files` |
   | `webSearch` | icon `magnifyingglass`, `narration.web_search` |
   | `todo` | icon `checklist`, `narration.todo_update` |
   | `message(text)` | none (final message arrives via the contract) |
   | `errorItem(message)` | icon `exclamationmark`, sanitized message ≤ 120 |
   | `unknown(type)` | none (logged upstream) |
   | `turnEnded` | none (FSM speaks results) |
   HUD lines are NOT throttled.
4. Spoken rules (encode DESIGN §5.3 exactly):
   - `quiet`: no item narration.
   - `milestones` (default): the first `commandExecution` with status
     `.running` per turn; thereafter at most one progress line per 20 s
     window where the LATEST candidate wins (candidate set:
     `commandExecution(.running)`, `webSearch`, `fileChange`); emission
     occurs on the first ingest after the window closes; the first
     `fileChange` with count > 0 is always spoken once.
   - `verbose`: every `commandExecution(.running)`, `webSearch`, `todo` —
     min 5 s spacing, latest-wins coalescing.
   - Pending coalesced narration is DISCARDED at `turnEnded` (results take
     over; stale progress is noise).
5. Sanitization: every agent-derived placeholder value (command text, file
   summary, web query, todo summary, error message) passes
   `SpeechTextSanitizer.sanitize` (spoken: command 60 chars word-boundary
   truncation per template; HUD: 120) BEFORE template rendering. Injection
   tests with SSML/control/URL payloads in command text.
6. `beginTurn` resets first-command flag, window state, and context.

## Acceptance Criteria

- [ ] Scripted timeline tests driving `now` explicitly: milestones speaks
      first-command + ≤ 1 line per 20 s window (latest wins) + one
      file-change; verbose respects 5 s spacing; quiet speaks none; a 5-burst
      of commands in one window yields exactly the last at rollover
      (exact expected spoken sequences asserted, not just counts).
- [ ] Escalation override tests (Req 2).
- [ ] HUD receives every mapped line in all modes; unmapped cases produce
      none (exhaustive switch test).
- [ ] Template rendering golden tests ja+en (60-char truncation, basename
      substitution via sanitizer).
- [ ] Injection corpus on command/query/error payloads → inert spoken text.
- [ ] Allowed-imports check: Foundation + HandsfreeAgent + HandsfreeSpeech
      (SpokenUtterance) only (documented grep in Validation).

## Validation

`swift test --filter NarrationPolicyTests`;
`grep -n "^import" Sources/HandsfreeCore/Dialogue/NarrationPolicy.swift`
output in PR.

## Dependencies

10 (SpokenUtterance/SpeechPriority), 12, 19, 20.

## Non-goals

Result/approval/error announcements (FSM/orchestrator), arbiter staleness
(11), HUD rendering (31).

## Design References

DESIGN.md §5.3, §4.4, §9.2 (T7); ADR-008.
