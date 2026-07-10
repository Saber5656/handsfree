# Title

VAD integration and utterance endpointing policy

## Summary

Implement `SpeechDetectorVAD` (composed into issue 08's `SpeechPipeline`),
the pure-Swift `RMSGateVAD` fallback, and the `EndpointingPolicy` state
machine that decides when an utterance is complete.

## Context

Endpointing quality defines the feel of continuous conversation (R6).
`SpeechDetector`'s runtime behavior is known unknown #1; the design mandates
a swappable fallback. Session-idle behavior (30 s timeout) belongs to the FSM
(issue 22), NOT to endpointing.

## Scope

- `HandsfreeSpeech/VAD/`: `SpeechDetectorVAD`, `RMSGateVAD`,
  `EndpointingPolicy`, `VADProviderFactory`. Not: STT decoding (08), FSM
  timers (22).

## Detailed Requirements

1. `SpeechDetectorVAD: VADProvider` — attaches a `SpeechDetector` module to
   the shared `SpeechPipeline` attachment point provided by issue 08 (public
   API of 08 unchanged). Emits `VADVerdict`s per the issue-07 semantics
   (silence carries contiguous-silence duration since last speech).
2. `RMSGateVAD: VADProvider` — deterministic energy gate, exact math:
   - samples normalized to [-1, 1] floats; rolling 100 ms windows;
   - linear RMS per window; noise floor = 5th percentile of the last 10 s of
     window RMS values, clamped to ≥ `10^(-60/20)` (linear equivalent of
     −60 dBFS); during the first 1 s (warm-up) the clamp value is used;
   - speech verdict while `rms > floor × 3`; hangover 300 ms (speech verdict
     persists through sub-threshold windows shorter than the hangover).
3. `EndpointingPolicy` (pure; consumed by 28):
   ```swift
   public struct EndpointingPolicy {
       public init(tuning: EndpointingTuning = .default)
       public mutating func ingest(at now: Duration,       // injected monotonic time
                                   vad: VADVerdict?,
                                   stt: STTResult?) -> EndpointDecision
       public mutating func reset()
   }
   public enum EndpointDecision: Equatable { case `continue`
       case finalize(reason: FinalizeReason) }   // .silence | .maxDuration
   public struct EndpointingTuning { silence: Duration = .milliseconds(900)
       maxUtterance: Duration = .seconds(60) }   // internal constants, not user config
   ```
   Rules (DESIGN §4.3): finalize when contiguous silence ≥ `silence` AND a
   final STT result has arrived since the last speech verdict; or when speech
   duration since the first speech verdict reaches `maxUtterance` (finalize
   + the caller notifies the user). No "abandon" concept here — silence with
   no speech at all simply keeps returning `continue`; the FSM's idle timer
   (issue 22) owns that case.
4. `VADProviderFactory`: prefers `SpeechDetectorVAD`; falls back to
   `RMSGateVAD` when SpeechDetector is unavailable or throws at
   construction (logged once). **Empirical gate (closes known unknown #1)**:
   run the live latency test below on both paths; if SpeechDetector fails
   the criterion (finalize within 1.5 s of stopping speech in 5/5 runs),
   the factory default flips to `RMSGateVAD` and DESIGN §16 #1 is updated
   with the evidence in the same PR — either way the unknown is closed.
5. VAD-verdict flapping (jitter shorter than hangover) is `RMSGateVAD`'s
   concern (hangover); `EndpointingPolicy` consumes post-hangover verdicts
   and applies no additional debounce (documented).

## Acceptance Criteria

- [ ] `EndpointingPolicy` table-driven tests (≥ 10): silence-finalize only
      after a final STT result; cap-finalize; long silence without any
      speech → `continue` forever; reset behavior; boundary at exactly
      900 ms.
- [ ] `RMSGateVAD` synthesized-audio tests: burst/gap patterns, floor
      adaptation over 10 s, warm-up clamp, hangover suppression of
      sub-300 ms gaps.
- [ ] Factory fallback engages on seam-injected SpeechDetector construction
      failure.
- [ ] `VADLiveTests` (manual): count-to-five utterance finalizes within
      1.5 s of stopping speech on the chosen default path; result of BOTH
      paths recorded; DESIGN §16 #1 updated with the decision.

## Validation

`swift test --filter 'EndpointingPolicyTests|RMSGateVADTests'` (CI-safe);
`swift test --filter VADLiveTests` locally — evidence in PR.

## Dependencies

06, 07, 08.

## Non-goals

Barge-in (ADR-008), wake word, user-configurable endpointing constants,
session idle timeout (22).

## Design References

DESIGN.md §4.3, §16 (#1); ADR-008; research doc (SpeechDetector).
