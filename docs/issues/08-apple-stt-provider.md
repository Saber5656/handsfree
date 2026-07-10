# Title

AppleSTTProvider: SpeechTranscriber streaming STT with asset management

## Summary

Implement the production `STTProvider` on the macOS 26 `SpeechAnalyzer` +
`SpeechTranscriber` stack: streaming volatile/final results for ja-JP and
en-US, locale asset install/download flow, and authorization handling.

## Context

Research (2026-07-08) confirmed `SpeechTranscriber` supports ja-JP/en-* fully
on-device on macOS 26, with per-locale downloadable assets via
`AssetInventory`. This provider is the default and only STT in v1 (ADR-003,
ADR-007). Known unknown #2: whether this path also requires legacy
speech-recognition TCC — this issue must pin it empirically.

## Scope

- `HandsfreeSpeech/STT/AppleSTTProvider.swift` + asset manager + errors.
- Not: endpointing (09), audio capture (06), onboarding UI (34).

## Detailed Requirements

1. Conform to `STTProvider` (07). Construction takes an
   `AudioEngineManager`-produced buffer stream (`AsyncStream<CapturedBuffer>`);
   internally convert buffers to the analyzer's expected input
   (`AnalyzerInput`), respecting the transcriber's `bestAvailableAudioFormat`
   (insert `AVAudioConverter` when formats differ).
2. `availability(locale:)`:
   - unsupported locale (not in `SpeechTranscriber.supportedLocales`) →
     `.unsupportedLocale`
   - supported but not in `installedLocales` → `.assetDownloadRequired(sizeEstimate?)`
   - mic/speech authorization missing → `.unauthorized`
   - else `.available`.
3. `prepare(locale:)`: request authorization(s) as required, then drive
   `AssetInventory` install with progress reporting via an
   `AsyncStream<Double>` exposed as `preparationProgress` (consumed by
   onboarding/Settings). Handle: no network (typed error), user cancel.
4. `startStream`: build `SpeechAnalyzer` with a `SpeechTranscriber` module
   configured for volatile + finalized results; map results to `STTResult`
   (`isFinal`, text, confidence when available, audio time range). Locale is
   fixed per stream (DESIGN §4.2; `auto` resolution happens in Core, not here).
5. `stopStream`: finalize the analyzer cleanly so trailing audio yields a
   final result before the stream ends (needed for endpointing correctness).
6. Empirical authorization pinning (**must be in the PR description**): with
   the ad-hoc bundle from issue 05, document which TCC prompts actually fire
   (mic only, or mic + speech recognition), and adjust `availability` +
   Info.plist docs accordingly; update DESIGN §16 known-unknown #2 as resolved
   in the same PR.
7. Errors (typed `STTError`): `notAuthorized`, `assetUnavailable`,
   `analyzerFailed(underlying)`, `formatUnsupported` — each mapped to a spoken
   error key later (28); log via `Log.stt`.
8. All tests that hit the real framework are `live`-tagged; pure logic
   (availability mapping, format conversion decision table) is unit-tested
   with seams.

## Acceptance Criteria

- [ ] `live` manual test: with ja-JP assets installed, speaking a 5-second
      Japanese sentence yields ≥1 volatile result and exactly one final result
      whose text is non-empty; same for en-US (after `prepare` downloads it).
- [ ] `availability` decision table unit-tested for all four outcomes.
- [ ] `stopStream` mid-utterance still delivers a trailing final result
      (`live` test).
- [ ] Authorization findings documented + DESIGN §16 updated (known unknown #2
      closed).
- [ ] No network use besides Apple's own asset download; no audio persisted.

## Validation

`swift test --filter AppleSTTAvailabilityTests`; `live` checklist run recorded
in the PR (device, locale states, transcript samples ja+en).

## Dependencies

06, 07.

## Non-goals

Custom vocabulary/biasing (known unknown #7 — separate issue if needed),
per-utterance language auto-detection, cloud STT.

## Design References

DESIGN.md §4.2, §12, §16 (#2, #7); ADR-003, ADR-007; research doc (speech stack).
