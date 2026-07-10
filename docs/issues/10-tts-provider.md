# Title

TTSProvider protocol, AppleTTSProvider, and MockTTSProvider

## Summary

Define the `TTSProvider` protocol per DESIGN §4.4 and implement the production
`AVSpeechSynthesizer` provider (voice selection with fallback, rate mapping,
event stream) plus a scriptable mock in `HandsfreeTestSupport`.

## Context

Only compact-quality voices are preinstalled (research doc probe); users may
delete or add voices anytime, so identifier-based selection must degrade
gracefully. The arbiter (11) consumes the event stream to gate STT
(half-duplex, ADR-008).

## Scope

- `HandsfreeSpeech/TTS/`: protocol + `AppleTTSProvider` + `VoiceDescriptor`
  catalog API; `MockTTSProvider` in TestSupport.
- Not: queueing/priorities (11), sanitization (20 — input text arrives
  pre-sanitized), Settings UI (32).

## Detailed Requirements

1. Protocol exactly per DESIGN §4.4 (`speak`, `stop`, `voices(for:)`,
   `SpokenUtterance`, `TTSEvent: .started/.finished/.cancelled`).
   `speak` returns its event stream; exactly one terminal event per utterance
   is guaranteed (`.finished` or `.cancelled`), even on synthesizer errors
   (error → `.cancelled` + `Log.tts` error).
2. `AppleTTSProvider`:
   - Voice resolution order: configured identifier for the utterance's
     language (`voice.tts_voice_ja` / `tts_voice_en`) → if missing/deleted,
     `AVSpeechSynthesisVoice(language:)` default → log + remember the
     degradation (surfaced by Settings later; expose
     `lastResolutionNote: String?`).
   - `voices(for:)` maps installed voices to `VoiceDescriptor{id, name,
     quality: .compact/.enhanced/.premium, language}` (quality from
     `AVSpeechSynthesisVoiceQuality`).
   - Rate: map config `speaking_rate` 0.1…1.0 linearly onto
     `AVSpeechUtteranceMinimumSpeechRate…Maximum`, default 0.5 ≈
     `AVSpeechUtteranceDefaultSpeechRate` (document the mapping formula in a
     code comment + unit test).
   - `stop()` = `stopSpeaking(at: .immediate)`; the pending utterance's stream
     must emit `.cancelled`.
   - One serial internal queue: `speak` while speaking is a programmer error
     in v1 (the arbiter serializes); assert + drop with log rather than crash.
3. `MockTTSProvider` (TestSupport): scripted durations on the shared
   `TestClock`; records spoken texts/locales; supports forced mid-utterance
   `stop()` assertions. This mock is what E2E tests read "speech" from.
4. Language hint: `SpokenUtterance.localeHint` selects the voice language;
   mixed-language text is spoken with the hint voice (no per-run detection in
   v1 — document).

## Acceptance Criteria

- [ ] Terminal-event guarantee test (finish, cancel, injected error) via a
      synthesizer seam.
- [ ] Voice fallback: configured-but-missing identifier degrades to language
      default and sets `lastResolutionNote` (unit test with seam).
- [ ] Rate mapping unit test at 0.1/0.5/1.0.
- [ ] `voices(for: ja)` returns Kyoko on a stock machine (`live` tag).
- [ ] `live` smoke: speaks a ja and an en sentence audibly with correct voices.
- [ ] Mock: scripted timing + cancellation behavior verified.

## Validation

`swift test --filter TTSProviderTests MockTTSProviderTests`; `live` checklist
notes (voices heard, ja/en) in PR.

## Dependencies

01 (TestSupport target exists after 07 — coordinate; if 07 not merged, this
issue creates it with the same spec).

## Non-goals

Priority queueing/flushing (11), SSML, cloud TTS, per-utterance language
detection, Personal Voice.

## Design References

DESIGN.md §4.4, §12; ADR-003, ADR-008; research doc (voice inventory probe).
