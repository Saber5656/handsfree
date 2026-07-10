# ADR-007: Minimum deployment target macOS 26.0

- Status: Accepted (2026-07-08)

## Context

The on-device STT that satisfies R3/R9 (`SpeechAnalyzer`/`SpeechTranscriber`,
ja-JP + en-* confirmed by probe) exists only on macOS 26+. The legacy
`SFSpeechRecognizer` (macOS 10.15+) has weaker continuous-dictation behavior
and a different lifecycle. GitHub Actions has GA macos-26 runners
(2026-02-26), so CI parity exists.

## Decision

v1 targets **macOS 26.0 or newer** exclusively. No `SFSpeechRecognizer`
fallback in v1; the `STTProvider` protocol keeps that door open for v2 if
demand appears.

## Rationale

- One STT runtime = one behavior matrix for endpointing, assets, and QA.
- macOS 26 has been the current major release since 2025-09; by v1 launch the
  install base among developer users (the target persona) is high.
- The fallback would double the audio test matrix for a strictly worse engine.

## Consequences

- Users on macOS ≤ 15 cannot run Handsfree v1; README states this prominently.
- `Package.swift` sets `platforms: [.macOS(.v26)]` (or the SwiftPM equivalent
  available at scaffold time); availability annotations are not scattered
  through code.
- v2 item recorded: `SFSpeechRecognizer` provider for macOS 15.
