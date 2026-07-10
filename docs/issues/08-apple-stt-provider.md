# Title

AppleSTTProvider: SpeechTranscriber streaming STT with asset management

## Summary

Implement the production `STTProvider` on the macOS 26 `SpeechAnalyzer` +
`SpeechTranscriber` stack: streaming volatile/final results for ja-JP and
en-US, locale asset install flow, authorization handling, and the shared
analyzer pipeline object that issue 09's VAD composes into.

## Context

Research (2026-07-08) confirmed on-device ja-JP/en-* support with per-locale
downloadable assets (`AssetInventory`). This is the only STT in v1 (ADR-003,
ADR-007). Known unknown #2 (whether legacy speech-recognition TCC is also
required) is pinned empirically here.

## Scope

- `HandsfreeSpeech/STT/`: `AppleSTTProvider`, `SpeechPipeline` (shared
  analyzer owner), asset manager, error taxonomy, seams. Not: endpointing
  policy (09), capture (06), onboarding UI (34).

## Detailed Requirements

1. Conform exactly to `STTProvider` (07). Internally, `startStream` builds a
   `SpeechPipeline` object that owns the `SpeechAnalyzer` and its modules;
   the pipeline exposes an attachment point for additional modules
   (`SpeechDetector`, issue 09) WITHOUT changing this provider's public API.
   Buffer conversion: consume `CapturedBuffer`s, convert via
   `AVAudioConverter` when the native format differs from the transcriber's
   `bestAvailableAudioFormat`, and feed `AnalyzerInput`s.
2. `availability(locale:)` decision table:
   not in `SpeechTranscriber.supportedLocales` → `.unsupportedLocale`;
   supported but not in `installedLocales` → `.assetDownloadRequired(size?)`;
   authorization missing → `.unauthorized`; else `.available`.
3. `prepare(locale:)`: request required authorization(s), then drive
   `AssetInventory` installation. Progress: expose
   `public var preparationProgress: AsyncStream<Double>` — single active
   consumer, emits 0…1 for the CURRENT prepare call, completes when prepare
   returns/throws (documented; issues 32/34 consume it).
4. `stopStream`: finalize the analyzer so trailing audio yields final
   result(s) before the stream ends (endpointing correctness).
5. Error taxonomy (complete): `STTError.notAuthorized`,
   `.assetUnavailable(locale)`, `.assetDownloadFailed(underlying)`,
   `.networkUnavailable`, `.cancelled`, `.analyzerFailed(underlying)`,
   `.formatUnsupported` — each with a spoken-error template key constant.
6. Seams (unit-testable decision logic): `SpeechAssetClient`
   (supported/installed/install), `SpeechAuthorizationClient` (status/
   request), `SpeechAnalyzerFactory` (pipeline construction). Fakes in
   TestSupport; the availability table and error mapping are tested with
   them. Real-stack tests live in `AppleSTTLiveTests` (issue-02 convention).
7. Privacy rule: transcript text is never logged as public payload — log
   metadata only (lengths, isFinal, locale). Raw audio is never persisted.
8. **Authorization pinning (closes known unknown #2)**: using the issue-05
   bundle, document in the PR which TCC prompts actually fire (mic only, or
   mic + speech recognition); update `availability`'s `.unauthorized` logic
   accordingly AND update DESIGN §16 #2 + research doc in the same PR.

## Acceptance Criteria

- [ ] Availability decision table: all four outcomes unit-tested via seams.
- [ ] Error mapping tests: asset download failure, no network, user cancel,
      analyzer failure → correct `STTError` cases.
- [ ] `AppleSTTLiveTests` (manual, from the assembled bundle): ja-JP
      5-second utterance yields ≥ 1 volatile result and ≥ 1 final result with
      non-empty aggregate final text; same for en-US after `prepare`
      downloads it; `stopStream` mid-utterance still delivers trailing final
      result(s).
- [ ] `preparationProgress` emits monotonically increasing values ending at
      1.0 on the live en-US download (or seam-simulated in CI).
- [ ] TCC findings documented; DESIGN §16 #2 and research doc updated.
- [ ] No public-privacy logging of transcript text (code review + grep note).

## Validation

`swift test --filter AppleSTTProviderTests` (seam-based, CI-safe);
`swift test --filter AppleSTTLiveTests` locally — checklist + transcript
samples (ja/en) in the PR.

## Dependencies

05, 06, 07.

## Non-goals

Custom vocabulary/biasing (known unknown #7 — new issue if needed),
per-utterance language auto-detection, cloud STT, VAD (09).

## Design References

DESIGN.md §4.2, §12, §16 (#2, #7); ADR-003, ADR-007; research doc.
