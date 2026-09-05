# Title

AgentAdapter protocol and AgentEvent domain model

## Summary

Define the agent-integration domain model in `HandsfreeAgent` exactly per
DESIGN §6.1 (post-review revision: single terminal event `turnEnded`),
including the full `AgentResponse` value type, plus a scriptable
`MockAgentAdapter` in `HandsfreeTestSupport`.

## Context

This protocol keeps v1 codex-only while enabling v2 adapters (ADR-002). The
FSM (22), narration (23), task manager (26), and orchestrator (28) are
written against these types. DESIGN §6.1 was amended so that every turn ends
with exactly one `turnEnded(TurnOutcome)` — there is no separate
`turnFailed` event.

## Scope

- Types/protocols + doc comments in `HandsfreeAgent`; `MockAgentAdapter` in
  the existing `HandsfreeTestSupport` target (01). No process execution (14),
  no JSONL (15), no parsing (16).

## Detailed Requirements

1. Exact declarations (mirroring DESIGN §6.1 as amended):
   ```swift
   public struct AgentAdapterID: RawRepresentable, Hashable, Sendable { public let rawValue: String }
   public enum RiskTier: Int, Comparable, Sendable, Codable {
       case t0Read, t1Workspace, t2Network, t3Full
       public var displayNameKey: String { get }
   }
   public struct TurnRequest: Sendable {
       public let projectPath: URL; public let prompt: String
       public let resumeThreadID: String?; public let tier: RiskTier
       public let model: String?; public let timeout: Duration
   }
   public struct ProcessIdentity: Equatable, Sendable, Codable {
       public let pid: Int32; public let processGroupID: Int32
       public let startTvSec: Int64; public let startTvUsec: Int64   // proc_bsdinfo fields
   }
   public protocol AgentAdapter: Sendable {
       var id: AgentAdapterID { get }
       func preflight() async -> AgentPreflight
       func startTurn(_ request: TurnRequest) throws -> RunningTurn
   }
   public final class RunningTurn: Sendable {
       public let events: AsyncThrowingStream<AgentEvent, Error>
       public let processIdentity: ProcessIdentity?
       public func cancel() async     // idempotent; events still end with one turnEnded
   }
   public enum AgentEvent: Sendable {
       case threadStarted(id: String)
       case turnStarted
       case item(AgentItem)
       case turnEnded(TurnOutcome)    // exactly once, always last
   }
   public enum ItemStatus: Sendable, Equatable { case running, succeeded, failed }
   public enum AgentItem: Sendable {
       case commandExecution(command: String, status: ItemStatus)
       case fileChange(summary: String, fileCount: Int?)
       case message(text: String)
       case webSearch(query: String?)
       case todo(summary: String?)
       case errorItem(message: String)      // NON-terminal (codex quirk)
       case unknown(type: String)
   }
   public enum TurnStatus: Sendable, Equatable {
       case completed
       case failed(reason: String)
       case cancelled
       case timedOut
   }
   public struct TurnUsage: Sendable, Equatable {
       public let inputTokens: Int?; public let cachedInputTokens: Int?
       public let outputTokens: Int?; public let reasoningOutputTokens: Int?
   }
   public struct TurnOutcome: Sendable {
       public let status: TurnStatus
       public let contract: AgentResponse?   // populated only for .completed (16 parses)
       public let rawFinalText: String?
       public let usage: TurnUsage?
       public let threadID: String?
   }
   public struct AgentResponse: Sendable, Equatable {   // fields per Appendix C.1
       public let status: ResponseStatus                 // .ok/.needsInput/.blocked/.failed
       public let voiceSummary: String
       public let question: String?
       public let blockedReason: BlockedReason?          // .needsNetwork/.needsFullAccess/.needsOutOfWorkspace
       public let blockedAction: String?
       public let detail: String?
       public let proposedNextAction: String?
   }
   public enum AuthState: Sendable, Equatable { case authenticated, notAuthenticated, unknown }
   public struct AgentPreflight: Sendable {
       public let binaryPath: String?; public let version: String?
       public let auth: AuthState
       public let problems: [PreflightProblem]   // BLOCKING — non-empty ⇒ no dispatch
       public let warnings: [PreflightWarning]   // non-blocking (e.g. untested version)
   }
   ```
   (`PreflightProblem`/`PreflightWarning` enum cases are owned by issue 13;
   declare them here as frozen-shape enums with a `case placeholder` removed
   by 13 — or simply declare the enums empty here and let 13 add cases;
   choose the latter and note it.)
2. Threading contract doc comments: adapters `Sendable`; events stream
   single-consumer; exactly one `turnEnded` per turn, always the final
   element before the stream finishes; `cancel()` idempotent.
3. `MockAgentAdapter` (file `Sources/HandsfreeTestSupport/MockAgentAdapter.swift`):
   ```swift
   public enum MockAgentScriptEvent { case emit(after: Duration, AgentEvent)
                                      case streamError(after: Duration, any Error) }
   public final class MockAgentAdapter: AgentAdapter {
       public init(preflight: AgentPreflight, script: [MockAgentScriptEvent], clock: TestClock)
       public private(set) var requests: [TurnRequest]
       public private(set) var cancelledTurnCount: Int
   }
   ```
   After `cancel()`, the mock stops the script and emits
   `turnEnded(.cancelled)` as the terminal event (matching the production
   contract).
4. Exhaustiveness guard: unit test switches over `AgentItem`/`TurnStatus`/
   `AgentEvent` without `default`.

## Acceptance Criteria

- [ ] Declarations compile in `HandsfreeAgent` with no Foundation.Process or
      codex-specific imports; shapes match this issue verbatim.
- [ ] Mock replays a scripted happy turn (thread→turn→items→turnEnded) and a
      cancelled turn (terminal `turnEnded(.cancelled)`), with request
      recording asserted.
- [ ] Exhaustiveness-guard tests present.

## Validation

`swift test --filter 'AgentModelTests|MockAgentAdapterTests'`.

## Dependencies

01.

## Non-goals

Codex execution (14/18), JSONL (15), contract parsing (16), preflight logic
(13).

## Design References

DESIGN.md §6.1 (amended), §6.5, Appendix C.1; ADR-002.
