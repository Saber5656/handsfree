# Title

STTProvider/VADProvider protocols, result types, and scriptable mocks

## Summary

Define the speech-to-text and voice-activity-detection provider protocols and
their concrete data types (per DESIGN §4.2/§4.3, exact declarations below),
plus deterministic mocks in `HandsfreeTestSupport`.

## Context

Provider abstraction is the load-bearing decision of ADR-003: Apple providers
now, cloud/whisper later, mocks for CI. The orchestrator (28) and golden-path
suite (38) are written against these protocols. The `HandsfreeTestSupport`
target and `TestClock` already exist (issue 01).

## Scope

- Protocols + types in `HandsfreeSpeech` (STT/, VAD/); `MockSTTProvider` and
  `MockVADProvider` in `HandsfreeTestSupport`. Not: Apple implementations
  (08/09), audio capture (06).

## Detailed Requirements

1. Exact declarations (deviations require a DESIGN.md §4.2 update in the
   same PR):
   ```swift
   public struct STTProviderID: RawRepresentable, Hashable, Sendable {
       public let rawValue: String            // "apple"
   }
   public enum STTAvailability: Equatable, Sendable {
       case available
       case assetDownloadRequired(estimatedBytes: Int64?)
       case unsupportedLocale
       case unauthorized
   }
   public protocol STTProvider: Sendable {
       var id: STTProviderID { get }
       func availability(locale: Locale) async -> STTAvailability
       func prepare(locale: Locale) async throws
       func startStream(locale: Locale, audio: AsyncStream<CapturedBuffer>)
           -> AsyncThrowingStream<STTResult, Error>
       func stopStream() async
   }
   public struct STTResult: Equatable, Sendable {
       public let text: String
       public let isFinal: Bool
       public let confidence: Double?
       public let audioRange: ClosedRange<TimeInterval>?
   }
   public enum VADVerdict: Equatable, Sendable {
       case speech
       case silence(total: Duration)   // contiguous silence measured since the
                                       // last speech verdict (provider-resettable)
   }
   public protocol VADProvider: Sendable {
       func process(_ buffer: CapturedBuffer) async -> VADVerdict
       func reset() async
   }
   ```
2. Contract doc comments: providers callable from any actor; streams finish
   on `stopStream`; providers are restartable (start→stop→start); exactly one
   terminal (finish or throw) per stream.
3. `MockSTTProvider` (TestSupport), fully specified:
   ```swift
   public enum MockSTTEvent { case result(after: Duration, STTResult)
                              case fail(after: Duration, any Error) }
   public final class MockSTTProvider: STTProvider {
       public init(script: [MockSTTEvent], clock: TestClock)
       public private(set) var startCalls: [(Locale)]
       public private(set) var stopAwaited: Bool
       public private(set) var preparedLocales: [Locale]
       public var availabilityByLocale: [String: STTAvailability]  // settable
   }
   ```
   Replays the script on the injected `TestClock`; `fail` throws out of the
   stream; restartable (a second `startStream` replays from the script head).
4. `MockVADProvider`: scripted verdicts keyed by buffer index
   (`init(verdicts: [VADVerdict], clock: TestClock)`, default `.speech`
   after script exhaustion), call recording, `reset()` counter.
5. Target-graph oracle: production targets must not depend on
   `HandsfreeTestSupport`; TestSupport may depend on Core/Speech/Agent; only
   test targets depend on TestSupport (same script as issue 01 — rerun).

## Acceptance Criteria

- [ ] Declarations compile exactly as specified (reviewer diff vs this
      issue; DESIGN §4.2 already matches).
- [ ] Mock STT replay: volatile→final ordering with TestClock timing; error
      injection surfaces as stream throw; restartability proven.
- [ ] Mock VAD: scripted sequence + exhaustion default + reset behavior.
- [ ] Target-graph assertion output pasted in PR.

## Validation

`swift test --filter 'MockSTTProviderTests|MockVADProviderTests'`.

## Dependencies

01, 06 (`CapturedBuffer`).

## Non-goals

Apple/live providers, endpointing policy (09), locale asset UI.

## Design References

DESIGN.md §4.2, §4.3, §11; ADR-003.
