# Title

NarrationPolicy: verbosity-tiered, throttled progress narration

## Summary

Implement `HandsfreeCore/Dialogue/NarrationPolicy`: convert the `AgentEvent`
stream into (a) throttled spoken narration lines and (b) unthrottled HUD lines,
per verbosity level.

## Context

DESIGN §5.3 fixes the verbosity table; ADR-008 makes throttling essential
(every spoken line blocks the mic in half-duplex). Stale-progress dropping is
shared with the arbiter (11): the policy throttles at *emission*, the arbiter
drops at *dequeue*.

## Scope

- Policy engine + templates usage + tests. Not: arbiter queueing (11),
  event production (18).

## Detailed Requirements

1. API:
   ```swift
   struct NarrationPolicy {
       init(verbosity: NarrationVerbosity, locale: SpeechLocale, clock: any Clock<Duration>)
       mutating func ingest(_ event: AgentEvent) -> NarrationOutput
   }
   struct NarrationOutput { let spoken: SpokenUtterance?; let hud: [HUDLine] }
   ```
2. HUD lines: EVERY visible event maps to a HUDLine (type icon key + text,
   sanitized via 20, command text truncated 120 chars) — no throttling.
3. Spoken rules per verbosity (DESIGN §5.3 table, encode exactly):
   - `quiet`: nothing from items (dispatch/result/approval/errors are spoken
     by the FSM layer, not here).
   - `milestones` (default): first `commandExecution` of the turn; then at
     most one progress line per 20 s where the LATEST candidate wins
     (coalescing buffer — emitting happens on the next ingest after the
     window, no internal timers so the type stays pure); first
     `fileChange` with count > 0.
   - `verbose`: every `commandExecution` start, `webSearch`, `todo` — min 5 s
     spacing, latest-wins coalescing.
4. Template rendering via phrase dictionaries (19): `narration.command`
   (`{command}` truncated 60 chars word-boundary), `narration.files`
   (`{count}`), `narration.web_search`, `narration.started`. Paths spoken as
   basenames (sanitizer rule, 20).
5. `reasoning`/`message` items are never narrated (the final message arrives
   via the contract, not narration). `errorItem` maps to a HUD line only
   (turn-level failures are FSM-spoken).
6. Turn lifecycle: `reset()` on turn start (clears first-command flag,
   coalescing state).

## Acceptance Criteria

- [ ] Slow-drip fixture (20 events / 200 ms, via scripted ingestion with
      TestClock): `milestones` speaks ≤ 2 progress lines + first-command +
      file-change; `verbose` respects 5 s spacing with latest-wins;
      `quiet` speaks none.
- [ ] HUD receives all 20 lines in every mode.
- [ ] Template rendering: ja and en outputs golden-tested (incl. 60-char
      truncation at word boundary and basename substitution).
- [ ] Coalescing: burst of 5 commands in one window → only the last is spoken
      at window rollover.
- [ ] Pure value type; no timers, no I/O (compile-time import check).

## Validation

`swift test --filter NarrationPolicyTests`.

## Dependencies

12, 19, 20.

## Non-goals

Result/approval/error speech (FSM/orchestrator), arbiter staleness dropping
(11), HUD rendering (31).

## Design References

DESIGN.md §5.3, §4.4; ADR-008.
