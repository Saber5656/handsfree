# Title

TTSProvider protocol, AppleTTSProvider, and MockTTSProvider

## Summary

Define the `TTSProvider` protocol and its types (exact declarations below,
per DESIGN §4.4) and implement the production `AVSpeechSynthesizer` provider
plus a scriptable mock in `HandsfreeTestSupport`.

## Context

Only compact-quality voices are preinstalled (research probe); users may
delete or add voices anytime, so identifier-based selection must degrade
gracefully. The arbiter (11) consumes the event stream for half-duplex
gating (ADR-008). Voice/rate settings are constructor-injected plain values —
config wiring happens in issues 28/32, keeping this issue free of the config
store.

## Scope

- `HandsfreeSpeech/TTS/`: protocol + types + `AppleTTSProvider`;
  `MockTTSProvider` in TestSupport. Not: queueing/priorities behavior (11),
  sanitization (20 — input text arrives sanitized), Settings UI (32).

## Detailed Requirements

1. Exact declarations:
   ```swift
   public enum SpeechPriority: Int, Comparable, Sendable {
       case narration = 0, result = 1, approval = 2   // data type here; queue semantics in 11
   }
   public struct SpokenUtterance: Sendable {
       public let text: String            // pre-sanitized upstream
       public let localeHint: Locale
       public let priority: SpeechPriority
   }
   public enum TTSEvent: Sendable { case started, finished, cancelled }
   public struct VoiceDescriptor: Equatable, Sendable {
       public let id: String; public let name: String
       public let quality: VoiceQuality   // .compact | .enhanced | .premium
       public let language: String        // BCP-47
   }
   public protocol TTSProvider: Sendable {
       func speak(_ u: SpokenUtterance) -> AsyncStream<TTSEvent>
       func stop()
       func voices(for locale: Locale) -> [VoiceDescriptor]
   }
   ```
2. `AppleTTSProvider`:
   - `init(settings: TTSSettings)` where
     `TTSSettings { voiceIDByLanguage: [String: String], rate: Double }` —
     plain values, updatable via `func apply(_ settings: TTSSettings)`.
   - Voice resolution: configured identifier for the utterance language →
     if missing/deleted, `AVSpeechSynthesisVoice(language:)` default →
     record `lastResolutionNote: String?` (surfaced by Settings).
   - Rate mapping: config 0.1…1.0 mapped linearly onto
     `[AVSpeechUtteranceMinimumSpeechRate, AVSpeechUtteranceMaximumSpeechRate]`
     with 0.5 → `AVSpeechUtteranceDefaultSpeechRate` (piecewise-linear around
     the default; formula in a comment + unit test at 0.1/0.5/1.0).
   - **Terminal-event guarantee**: every `speak` stream emits exactly one of
     `.finished`/`.cancelled` then finishes — including: `stop()` calls,
     synthesizer seam errors (mapped to `.cancelled` + log), AND the overlap
     case: `speak` while already speaking immediately returns a stream that
     emits `.cancelled` and finishes, plus logs the programmer error (the
     arbiter serializes; debug builds may additionally assert).
   - Synthesizer seam: `protocol SpeechSynthesizing` wrapping
     `AVSpeechSynthesizer` (speak/stopImmediate/delegate callbacks) with a
     TestSupport fake that can emit start/finish/cancel and a simulated
     failure (note in code: the real `AVSpeechSynthesizer` has no failure
     delegate; the seam models defensive handling only).
   - `voices(for:)` maps installed voices with quality from
     `AVSpeechSynthesisVoiceQuality`.
3. `MockTTSProvider` (TestSupport): `init(clock: TestClock,
   utteranceDuration: @Sendable (SpokenUtterance) -> Duration)`; records
   `(text, localeHint, priority, startedAt)`; `stop()` cancels the active
   utterance mid-flight; terminal-event guarantee identical to production.
   This mock is the "spoken transcript" source for E2E tests.
4. Language hint selects the voice language; mixed-language text speaks with
   the hint voice (no per-run detection in v1 — documented).

## Acceptance Criteria

- [ ] Terminal-event guarantee tests: finish, external stop, seam error,
      overlap-speak — each exactly one terminal event, stream finishes.
- [ ] Voice fallback sets `lastResolutionNote` (seam test).
- [ ] Rate mapping unit test at 0.1 / 0.5 / 1.0.
- [ ] `TTSLiveTests` (manual): `voices(for: ja)` includes Kyoko on a stock
      machine; speaks one ja and one en sentence audibly with the selected
      voices.
- [ ] Mock: scripted duration timing, cancellation recording, terminal
      guarantee.

## Validation

`swift test --filter 'AppleTTSProviderTests|MockTTSProviderTests'` (CI-safe);
`swift test --filter TTSLiveTests` locally — notes in PR.

## Dependencies

01, 07 (TestSupport target and TestClock exist via 01; 07 establishes the
mock conventions this issue follows).

## Non-goals

Priority queueing/flushing (11), SSML, cloud TTS, per-utterance language
detection, Personal Voice, config-store integration (28/32).

## Design References

DESIGN.md §4.4, §12; ADR-003, ADR-008; research doc (voice inventory probe).
