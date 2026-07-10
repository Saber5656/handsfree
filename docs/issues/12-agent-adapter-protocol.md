# Title

AgentAdapter protocol and AgentEvent domain model

## Summary

Define the agent-integration domain model in `HandsfreeAgent`: `AgentAdapter`,
`TurnRequest`, `RunningTurn`, `AgentEvent`, `AgentItem`, `TurnOutcome`,
`RiskTier`, `AgentPreflight` — exactly per DESIGN §6.1, with no codex-specific
code.

## Context

This protocol is the seam that keeps v1 codex-only while enabling v2 adapters
(ADR-002). The dialogue FSM (22), narration (23), task manager (26), and
orchestrator (28) are all written against these types, so they must land
before wave-2 implementation issues and stay stable.

## Scope

- Types/protocols + doc comments + a `MockAgentAdapter` in
  `HandsfreeTestSupport`. No process execution (14), no JSONL (15).

## Detailed Requirements

1. Implement DESIGN §6.1 signatures verbatim. Clarifications to encode:
   - `RiskTier` enum: `.t0Read, .t1Workspace, .t2Network, .t3Full` with
     `var displayNameKey: String` and a doc table referencing DESIGN §6.5.
   - `AgentItem` cases and payloads:
     `commandExecution(command: String, status: ItemStatus)`,
     `fileChange(summary: String, fileCount: Int?)`,
     `message(text: String)`, `webSearch(query: String?)`,
     `todo(summary: String?)`, `errorItem(message: String)`,
     `unknown(type: String)`. Payload strings are raw (sanitization happens at
     the speech boundary, issue 20).
   - `TurnOutcome`: `{ status: TurnStatus, contract: AgentResponse?,
     rawFinalText: String?, usage: TurnUsage?, threadID: String }` where
     `TurnStatus = .completed | .failed(reason: String) | .cancelled |
     .timedOut`. (`AgentResponse` is defined in issue 16; forward-declare via
     protocol `AgentResponseCarrying` or land the struct shell here with
     parsing in 16 — choose the struct-shell approach and document it.)
   - `RunningTurn.cancel()` semantics: idempotent; events stream MUST finish
     with a terminal event after cancel.
   - `AgentPreflight`: `{ binaryPath: String?, version: String?,
     versionSupported: Bool?, auth: AuthState (.authenticated | .notAuthenticated
     | .unknown), problems: [PreflightProblem] }`.
2. Threading contract in doc comments: adapters are `Sendable`; the events
   stream is single-consumer; all callbacks are async-stream based (no
   delegates).
3. `MockAgentAdapter` (TestSupport): scripted event sequences on `TestClock`,
   scriptable preflight results, records `TurnRequest`s, supports
   mid-stream cancellation assertions. This mock drives FSM tests (22) before
   the real adapter (18) exists.
4. Exhaustiveness guard: a unit test switches over `AgentItem`/`TurnStatus`
   without `default` so new cases force test updates.

## Acceptance Criteria

- [ ] Types compile in `HandsfreeAgent` with zero Foundation.Process /
      codex-specific imports.
- [ ] Signatures match DESIGN §6.1 (reviewer side-by-side; deviations update
      DESIGN.md in the same PR).
- [ ] `MockAgentAdapter` replays a scripted happy-path turn and a cancelled
      turn with correct terminal events (tests included).
- [ ] Exhaustiveness-guard tests present.

## Validation

`swift test --filter AgentModelTests MockAgentAdapterTests`.

## Dependencies

01.

## Non-goals

Codex execution, JSONL parsing, contract parsing, tier→flag mapping (18).

## Design References

DESIGN.md §6.1, §6.5; ADR-002.
