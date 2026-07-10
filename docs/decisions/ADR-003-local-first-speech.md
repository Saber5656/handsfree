# ADR-003: Local-first speech with provider abstraction

- Status: Accepted (2026-07-08, confirmed with product owner)

## Context

STT/TTS can run on-device (Apple Speech stack, whisper.cpp) or via cloud APIs
(higher accuracy/naturalness, but API keys, cost, and transmission of spoken
code/business context off-device).

## Decision

STT and TTS sit behind `STTProvider`/`TTSProvider` protocols. v1 ships only
Apple on-device providers (`SpeechTranscriber` STT, `AVSpeechSynthesizer` TTS).
Cloud providers are v2 opt-ins. The v1 app performs **zero network calls of its
own** and stores **no API credentials**.

## Rationale

- Verified on-device ja-JP + en-* STT support on macOS 26 (research
  2026-07-08) removes the historical accuracy argument for cloud STT defaults.
- Voice sessions contain source code, repo names, and business context;
  local-by-default is the only defensible OSS posture (threat T5).
- No key management in v1 dramatically shrinks the threat model and onboarding.

## Consequences

- Default TTS voices are compact quality; onboarding must guide users to
  download enhanced voices (probe evidence in research doc).
- Provider protocols must be designed now to fit streaming cloud backends
  later (async streams, explicit locale/asset lifecycle).
- Requires macOS 26 (ADR-007).
