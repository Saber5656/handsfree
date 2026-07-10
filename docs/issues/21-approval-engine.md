# Title

ApprovalEngine: nonce generation, echo verification, tiered decision flow

## Summary

Implement `HandsfreeCore/Approval/ApprovalEngine`: the security-critical state
machine that announces a pending escalation, verifies the spoken nonce echo,
applies Tier-3 screen confirmation, enforces timeouts/retries, and records an
audit trail.

## Context

ADR-004 and DESIGN §5.4 define this component; threats T1/T2/T3 all converge
here. It must be pure logic (TestClock-driven) with effects delegated, so the
adversarial suite can run in CI.

## Scope

- Engine + types + audit records + tests. Not: TTS/arbiter plumbing (28),
  HUD button (31), tier execution (18).

## Detailed Requirements

1. API:
   ```swift
   actor ApprovalEngine {
       func begin(request: ApprovalRequest) -> AsyncStream<ApprovalEffect>
       func handle(_ input: ApprovalInput) async
   }
   struct ApprovalRequest { let taskID: TaskID; let tier: RiskTier /* .t2Network|.t3Full only */;
                            let action: String /* sanitized upstream */; let locale: SpeechLocale }
   enum ApprovalInput { case utterance(String)          // routed via IntentMatcher by caller? NO — see Req 3
                        case screenConfirm(Bool)
                        case cancelled }
   enum ApprovalEffect { case speak(templateKey: String, args: [String:String], earcon: Earcon?)
                         case awaitEcho
                         case showScreenConfirm(action: String)
                         case decided(ApprovalDecision) }
   enum ApprovalDecision { case approved(tier: RiskTier)
                           case denied(reason: DenialReason) } // .userDenied|.timeout|.echoFailed|.cancelled
   ```
2. Nonce: two digits drawn independently from
   `SystemRandomNumberGenerator` (0–9 each). A nonce is **single-use**: it is
   invalidated on decision, timeout, or each failed attempt (a NEW nonce is
   generated for the retry announcement — replaying a previously heard nonce
   must never work).
3. Echo verification: the engine receives the RAW final utterance text and
   verifies via `IntentMatcher` in `approve_echo`-only context (dependency on
   19). Rules: exact keyword + exact both digits in order; any extra trailing
   tokens ⇒ failure (matcher guarantees this; engine adds a defense-in-depth
   re-check comparing extracted digits to the active nonce).
4. Flow (DESIGN §5.1/§5.4): `begin` →
   `.speak(approval.announce, {action,d1,d2}, earcon: .approvalRequest)` →
   `.awaitEcho` (timeout 20 s on injected clock) →
   - success + tier `.t2Network` → `.decided(.approved(.t2Network))`;
   - success + tier `.t3Full` → `.showScreenConfirm` (timeout 60 s) →
     confirm(true) → approved; confirm(false)/timeout → denied;
   - failed echo → `.speak(approval.retry …)` with NEW nonce, max 2 attempts
     total, then denied(.echoFailed);
   - `deny` intent or `.cancelled` at any point → denied immediately.
5. Config: `policy.tier3_screen_confirm=false` ⇒ t3 behaves like t2 (echo
   only) — engine reads a policy snapshot passed in the request context, not
   global config (testability). `policy.allow_tier3=false` is enforced
   UPSTREAM (the orchestrator never begins a t3 approval; assert here anyway:
   `begin` with disallowed tier → immediate denied(.userDenied) + log).
6. Audit record emitted with the decision (consumed by 27): request, nonce
   values,每 attempt's matched text (redacted via 04), decision, timestamps.
7. Adversarial test suite (CI, TestClock): wrong digits; one digit; reversed
   digits; correct digits after timeout; previous nonce replay on retry;
   extra-token suffix; deny word; silence to timeout; screen-confirm timeout;
   t3-with-screen-confirm-disabled path; 1000-iteration randomness sanity
   (both digits vary; no fixed seed leakage).

## Acceptance Criteria

- [ ] Every flow branch of Req 4 covered by a test (happy t2, happy t3, all
      denial paths).
- [ ] Nonce single-use and per-retry regeneration proven by tests.
- [ ] Adversarial suite green; audit record contents asserted.
- [ ] Engine is UI/TTS-free (only effects) and fully deterministic under
      TestClock except nonce values.

## Validation

`swift test --filter ApprovalEngineTests` (includes the adversarial suite).

## Dependencies

19, 20 (action text arrives sanitized; engine asserts length ≤ 300).

## Non-goals

Choosing WHEN approval is needed (orchestrator maps `blocked_reason` → tier,
28), rendering the HUD sheet (31), executing the escalated turn (18).

## Design References

DESIGN.md §5.4, §5.1, §6.5, §9.2 (T1/T2/T3); ADR-004.
