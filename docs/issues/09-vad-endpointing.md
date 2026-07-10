# Title

VAD integration and utterance endpointing policy

## Summary

Implement `SpeechDetectorVAD` (Speech framework `SpeechDetector`) plus the
`EndpointingPolicy` state machine that decides when an utterance is complete,
with the RMS-gate fallback specified in DESIGN §4.3.

## Context

Endpointing quality defines the feel of "continuous conversation" (R6). The
`SpeechDetector` module's runtime behavior is known unknown #1; the design
mandates a swappable fallback so the orchestrator never depends on which VAD
is active.

## Scope

- `HandsfreeSpeech/VAD/`: `SpeechDetectorVAD`, `RMSGateVAD` (fallback),
  `EndpointingPolicy`. Not: STT itself (08), FSM (22).

## Detailed Requirements

1. `SpeechDetectorVAD: VADProvider` — compose `SpeechDetector` into the
   analyzer pipeline (coordinate with 08's analyzer ownership: the analyzer is
   built once per stream with both modules; expose the VAD verdict stream from
   the shared pipeline object introduced in 08 — refactor there if needed, in
   this PR, keeping 08's public API stable).
2. `RMSGateVAD: VADProvider` — pure-Swift energy gate: rolling 100 ms RMS
   windows, speech if RMS > noise floor × 3 (floor = 5th percentile of last
   10 s, min −60 dBFS), hangover 300 ms. Deterministic, unit-testable with
   synthesized buffers (sine bursts + silence).
3. `EndpointingPolicy` (pure logic, lives here, consumed by 28):
   ```swift
   mutating func ingest(vad: VADVerdict, stt: STTResult?) -> EndpointDecision
   // .continue | .finalize(reason: .silence | .maxDuration) | .abandon(reason: .noSpeech)
   ```
   Rules (DESIGN §4.3): finalize when (VAD silence ≥ 900 ms AND a final STT
   result exists since last speech) OR 60 s hard cap; abandon if no speech
   within 10 s of listen start (feeds idle-timeout handling in 22).
   Constants in one struct `EndpointingTuning` with the defaults above
   (internal, not user config in v1).
4. Selection: `VADProviderFactory` chooses SpeechDetector when available,
   else RMS gate; a config-free automatic fallback if SpeechDetector init
   throws at runtime (log warning once).
5. Document (code comment + PR) the empirical findings on SpeechDetector
   cadence/latency; update DESIGN §16 known-unknown #1 status in the PR.

## Acceptance Criteria

- [ ] `EndpointingPolicy` table-driven tests: silence-finalize, cap-finalize,
      abandon, VAD flapping (speech/silence jitter < hangover) — ≥ 10 cases.
- [ ] `RMSGateVAD` synthesized-audio tests: burst/gap patterns yield expected
      verdict sequences; noise-floor adaptation verified.
- [ ] `live` test: real mic, count-to-five utterance finalizes within 1.5 s of
      stopping speech using SpeechDetector path; same passes with RMS fallback
      forced.
- [ ] Fallback engages automatically when SpeechDetector construction is
      forced to fail (seam-injected error).
- [ ] DESIGN known-unknown #1 updated with findings.

## Validation

`swift test --filter EndpointingPolicyTests RMSGateVADTests`; `live` run notes
in PR (both VAD paths, ja+en).

## Dependencies

06, 07 (and touches 08's pipeline object).

## Non-goals

Barge-in (ADR-008), wake word, tunable UI for endpointing constants.

## Design References

DESIGN.md §4.3, §16 (#1); ADR-008; research doc (SpeechDetector existence).
