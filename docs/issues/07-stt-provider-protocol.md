# Title

STTProvider/VADProvider protocols, result types, and scriptable mocks

## Summary

Define the speech-to-text and voice-activity-detection provider protocols and
data types exactly as DESIGN §4.2/§4.3, plus deterministic mock
implementations in a new `HandsfreeTestSupport` target used by all later
orchestrator and E2E tests.

## Context

Provider abstraction is the load-bearing decision of ADR-003: Apple providers
now, cloud/whisper later, mocks for CI. The orchestrator (28) and golden-path
suite (38) are written entirely against these protocols.

## Scope

- Protocols + types in `HandsfreeSpeech` (STT/, VAD/).
- New SwiftPM target `HandsfreeTestSupport` (library) added to `Package.swift`,
  depended on **only by test targets**; contains `MockSTTProvider`,
  `MockVADProvider` (and later mocks from issues 10/17).
- Not: Apple implementations (08/09), audio capture (06).

## Detailed Requirements

1. Types/protocols exactly per DESIGN §4.2 (`STTProvider`, `STTResult`,
   `STTAvailability`, `STTProviderID`) — copy the signatures; deviations
   require a DESIGN.md update in the same PR.
2. `VADProvider` protocol:
   ```swift
   public protocol VADProvider: Sendable {
       func process(_ buffer: CapturedBuffer) async -> VADVerdict // .speech | .silence(duration)
       func reset() async
   }
   ```
   (Endpointing *policy* is issue 09; this is the raw verdict source.)
3. `MockSTTProvider`: initialized with a script
   `[(delay: Duration, result: STTResult)]`; `startStream` replays the script
   on a test clock; supports mid-stream error injection and cancellation
   assertions (records whether `stopStream` was awaited).
4. `MockVADProvider`: scripted verdicts keyed by buffer index.
5. Both mocks record every call (`calls: [CallRecord]`) for behavioral asserts.
6. Test clock: use `swift-testing` + a small `TestClock` utility in
   `HandsfreeTestSupport` (hand-rolled, ~50 lines, no third-party dep) —
   shared by FSM/timeout tests in later issues (21/22/26).
7. Doc comments state threading/Sendable expectations: providers may be called
   from any actor; streams finish on `stopStream`; providers must be
   restartable (start→stop→start).

## Acceptance Criteria

- [ ] Protocol signatures compile and match DESIGN §4.2 verbatim (reviewer
      checks side-by-side; any change is reflected in DESIGN.md in the PR).
- [ ] `HandsfreeTestSupport` is NOT a dependency of any production target
      (assert via `swift package describe` in Validation).
- [ ] Mock STT replay test: scripted volatile→final sequence arrives in order
      with test-clock timing; error injection surfaces as stream throw.
- [ ] Restartability test passes for both mocks.
- [ ] `TestClock` supports advance/sleep semantics used by at least one
      example timeout test in this PR.

## Validation

`swift test --filter MockSTTProviderTests` (and VAD equivalent);
`swift package describe --type json` asserting TestSupport edges.

## Dependencies

01.

## Non-goals

Apple/live providers, endpointing policy, locale asset management.

## Design References

DESIGN.md §4.2, §4.3, §11; ADR-003.
