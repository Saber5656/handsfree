# Title

ApprovalEngine: nonce generation, echo verification, tiered decision flow

## Summary

Implement `HandsfreeCore/Approval/ApprovalEngine`: the security-critical
state machine that announces a pending escalation, verifies the spoken nonce
echo, applies Tier-3 screen confirmation, enforces timeouts/retries, and
emits an audit record with every decision.

## Context

ADR-004 / DESIGN §5.4 / threats T1–T3. Pure logic with injected clock and
nonce source so the adversarial suite runs deterministically in CI. Approval
announcements cap the action text at **120 chars** (DESIGN §5.4/§6.3).

## Scope

- Engine + types + audit records + tests. Not: TTS/arbiter plumbing (28),
  HUD buttons (31), escalated execution (18), transcript persistence (27 —
  the audit record is handed to the caller).

## Detailed Requirements

1. API:
   ```swift
   public struct ApprovalPolicySnapshot: Sendable {
       public let allowTier3: Bool
       public let tier3ScreenConfirm: Bool
   }
   public struct ApprovalRequest: Sendable {
       public let taskID: TaskID
       public let tier: RiskTier              // .t2Network or .t3Full only
       public let action: String              // sanitized upstream; engine re-truncates to 120
       public let locale: SpeechLocale
       public let policy: ApprovalPolicySnapshot
   }
   public enum ApprovalInput: Sendable {
       case utterance(String)                 // RAW final utterance text
       case screenConfirm(Bool)
       case cancelled
   }
   public struct ApprovalAuditRecord: Sendable {
       public let request: ApprovalRequest
       public let nonces: [(d1: Int, d2: Int)]        // one per announcement
       public let attempts: [String]                  // raw matched texts (redaction at persist time, 27)
       public let decision: ApprovalDecision
       public let startedAt: Duration; public let decidedAt: Duration   // engine clock
   }
   public enum ApprovalDecision: Sendable, Equatable {
       case approved(tier: RiskTier)
       case denied(reason: DenialReason)      // .userDenied | .timeout | .echoFailed | .cancelled | .tierDisabled
   }
   public enum ApprovalEffect: Sendable {
       case speak(templateKey: String, args: [String: String], earcon: Earcon?)
       case stageChanged(ApprovalStage)       // .announce | .awaitEcho | .awaitScreenConfirm
       case decided(ApprovalDecision, audit: ApprovalAuditRecord)
   }
   public protocol NonceGenerating: Sendable { func twoDigits() -> (Int, Int) }
   public actor ApprovalEngine {
       public init(matcher: IntentMatcher, nonce: any NonceGenerating = SystemNonceGenerator(),
                   clock: any Clock<Duration>)
       public func begin(_ request: ApprovalRequest) -> AsyncStream<ApprovalEffect>
       public func handle(_ input: ApprovalInput) async
   }
   ```
2. Nonce: two independent digits 0–9 from the injected generator
   (production `SystemNonceGenerator` on `SystemRandomNumberGenerator`).
   **Single-use**: invalidated on decision, timeout, and each failed
   attempt; every retry announcement carries a NEW nonce; a previously
   announced nonce must never verify.
3. Echo verification: `.utterance` texts are matched via the injected
   `IntentMatcher` with active set `[.approve_echo, .deny]` (deny stays
   available during echo wait; cancel arrives as `.cancelled`). On an
   `approve_echo(d1,d2)` result the engine re-checks the digits against the
   ACTIVE nonce (defense in depth). Non-matching utterances count as failed
   attempts.
4. Flow: `begin` → `.stageChanged(.announce)` +
   `.speak(approval.announce, {action,d1,d2}, earcon: .approvalRequest)` →
   `.stageChanged(.awaitEcho)` (20 s timeout):
   - echo OK + `.t2Network` → decided approved(t2).
   - echo OK + `.t3Full` + `tier3ScreenConfirm` → `.stageChanged(.awaitScreenConfirm)`
     (60 s) → `screenConfirm(true)` → approved(t3); `false`/timeout →
     denied(.userDenied)/(.timeout).
   - echo OK + `.t3Full` + screen-confirm disabled → approved(t3) directly.
   - failed echo → `.speak(approval.retry …)` with NEW nonce; max 2
     attempts total, then denied(.echoFailed).
   - `deny` intent / `.cancelled` anywhere → denied(.userDenied)/(.cancelled).
   - `begin` with `.t3Full` while `allowTier3 == false` → immediate
     `.speak(approval.tier3_disabled …)` + denied(.tierDisabled) (upstream
     should not request it — this is the engine-side guarantee).
5. Action text: engine truncates `request.action` to 120 chars via
   `SpeechTextSanitizer.truncate` before templating (belt-and-braces; input
   arrives sanitized).
6. Adversarial suite (CI, TestClock + fixed-sequence nonce generator):
   wrong digits; one digit; reversed; correct-after-timeout; PREVIOUS nonce
   on retry; extra-token suffix; deny word; silence to timeout;
   screen-confirm deny + timeout; tier3-with-confirm-disabled; tierDisabled
   path; separate randomness sanity test (1000 draws with the production
   generator: both digits vary, no fixed sequence).

## Acceptance Criteria

- [ ] Every flow branch of Req 4 has a test (happy t2, happy t3 both
      confirm modes, all six denial reasons).
- [ ] Nonce single-use + per-retry regeneration proven.
- [ ] Audit record contents asserted (nonces list length == announcements,
      attempts captured, timestamps from the test clock).
- [ ] Adversarial suite green; deny works during await_echo.
- [ ] Engine emits effects only (no TTS/UI/persistence imports).

## Validation

`swift test --filter ApprovalEngineTests`.

## Dependencies

19, 20.

## Non-goals

Deciding WHEN approval is needed (28 maps `blocked_reason` → tier and checks
`allow_tier3` upstream), HUD rendering (31), escalated-turn execution (18),
persistence (27).

## Design References

DESIGN.md §5.4 (amended: pointer-only screen confirm), §5.1, §6.5,
§9.2 (T1/T2/T3); ADR-004.
