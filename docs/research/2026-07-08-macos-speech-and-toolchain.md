# Research: macOS speech stack and build toolchain constraints

- Date: 2026-07-08
- Verified against: macOS **26.5.1** (Build 25F80), Swift **6.3.2** (Command Line Tools,
  no full Xcode installed), Apple SDK macosx26.x
- Method: compiled and executed a Swift probe against the real SDK on the dev machine
- Status: facts pinned for DESIGN.md §Speech subsystem and §Build & release

## STT: SpeechAnalyzer / SpeechTranscriber (macOS 26+)

Probe (compiled and run 2026-07-08 on this machine):

```
TRANSCRIBER_SUPPORTED: de-AT,de-CH,de-DE,en-AU,en-CA,en-GB,en-IE,en-IN,en-NZ,en-SG,
en-US,en-ZA,es-CL,es-ES,es-MX,es-US,fr-BE,fr-CA,fr-CH,fr-FR,it-CH,it-IT,ja-JP,ko-KR,
pt-BR,pt-PT,yue-CN,zh-CN,zh-HK,zh-TW
TRANSCRIBER_INSTALLED: ja-JP
SPEECH_DETECTOR_TYPE: SpeechDetector
```

Findings:

1. **`SpeechTranscriber` (new SpeechAnalyzer family) is available on macOS 26 and
   supports both `ja-JP` and `en-*` locales on-device.** This satisfies the
   ja/en bilingual requirement with zero cloud dependency and zero API keys.
2. **Model assets are per-locale downloads** managed via `AssetInventory`.
   On the dev machine `ja-JP` is already installed; `en-US` would be downloaded
   on first use. Onboarding/Settings must expose install state and trigger
   downloads explicitly (asset download requires network; STT itself is on-device).
3. **`SpeechDetector` exists in the Speech framework** (compile-time confirmed).
   It is the framework-native voice-activity detection module that composes with
   `SpeechAnalyzer`. The VAD/endpointing issue must validate its exact runtime
   behavior (result cadence, sensitivity knobs) as a first implementation step,
   with a fallback endpointing strategy based on transcriber volatile-result
   timing if `SpeechDetector` proves insufficient.
4. Authorization: microphone permission (`NSMicrophoneUsageDescription`) is
   definitely required. Whether the SpeechAnalyzer path additionally requires
   the legacy speech-recognition authorization prompt must be verified
   empirically in the STT issue (`NSSpeechRecognitionUsageDescription` is cheap
   to include regardless — it is only shown if the API demands it).

Implication: **minimum deployment target = macOS 26.0** (see ADR-007). A
`SFSpeechRecognizer`-based provider for macOS 15 is possible behind the same
protocol but is deferred to v2.

## TTS: AVSpeechSynthesizer voice inventory (probe evidence)

The probe enumerated installed voices for `ja` / `en-US` / `en-GB`:

- Only **compact/default-quality voices (quality raw value 1)** are present by
  default: `Kyoko` (ja-JP), `Samantha` (en-US), `Daniel` (en-GB), plus the
  Eloquence/novelty set. No enhanced (2) or premium (3) voices preinstalled.
- Enhanced/Premium voices (e.g., ja-JP `O-Ren (Premium)`, en-US `Ava (Premium)`)
  are user-downloadable via System Settings → Accessibility → Spoken Content →
  System Voice → Manage Voices. They are then selectable via
  `AVSpeechSynthesisVoice(identifier:)`.
- Siri voices are **not** exposed to third-party apps via `AVSpeechSynthesizer`.

Product implications:

1. Default out-of-box TTS quality is mediocre (compact Kyoko/Samantha).
   Onboarding must include a "download an enhanced voice" step with a deep link
   to the Settings pane, and the voice picker must show quality labels.
2. The TTS provider protocol must select by voice identifier with fallback to
   `AVSpeechSynthesisVoice(language:)` when the configured identifier is missing
   (voice can be deleted by the user at any time).
3. Cloud TTS providers remain a v2 opt-in; no key handling in v1.

## Build toolchain constraint: no full Xcode on the dev machine

Verified on the dev machine:

| Tool | Status |
|---|---|
| `xcodebuild` | **absent** (`xcode-select` points to CommandLineTools) |
| `swift` (SwiftPM) | present, Swift 6.3.2, target arm64-apple-macosx26.0 |
| `actool` (asset catalogs) | **absent** |
| `codesign` | present (`/usr/bin/codesign`) |
| `notarytool` | present (`/Library/Developer/CommandLineTools/usr/bin/notarytool`) |
| `iconutil` | present (`/usr/bin/iconutil`) |

Consequences (pinned as ADR-006):

1. The project must build as a **pure SwiftPM package** (`Package.swift`), no
   `.xcodeproj`. Implementation agents can then work with `swift build` /
   `swift test` only.
2. The `.app` bundle is assembled by a **script** (`scripts/make-app.sh`):
   copy binary into `Handsfree.app/Contents/MacOS/`, write `Info.plist`
   (usage strings, LSUIElement), copy resources, generate `.icns` via
   `iconutil` from pre-rendered PNGs (no asset catalog, since `actool` is
   unavailable).
3. Signing (`codesign` with entitlements + hardened runtime) and notarization
   (`xcrun notarytool submit … --wait`, then `stapler`) work with CLT alone —
   full Xcode is NOT required for the v1 release pipeline.
4. TCC behavior: microphone/speech permission prompts attribute to the app
   bundle. Dev loop uses the assembled `.app`; bare `swift run` executables
   attribute TCC to the invoking terminal and must not be used for
   permission-dependent testing (documented in CONTRIBUTING).

## CI: GitHub Actions runner support

- `macos-26` (arm64) runner image: public preview 2025-09-11, **generally
  available since 2026-02-26**. `macos-latest` maps to macos-26 from June 2026.
- Xcode 26.x is the default toolchain on that image, so CI can build the
  SwiftPM package and run all mock-based unit tests on macOS 26 APIs.
- CI runners have no microphone and should not download speech model assets:
  audio-path and live-STT tests are excluded from CI (tagged manual/local),
  which is why FakeCodex + mock speech providers are the CI backbone.

## Sources

- Swift probe compiled/run on this machine, 2026-07-08 (output quoted above)
- `xcrun --find notarytool`, `xcrun --find actool`, `which codesign iconutil`, 2026-07-08
- Apple Developer documentation: SpeechAnalyzer / SpeechTranscriber / Speech framework
  (https://developer.apple.com/documentation/speech/speechanalyzer,
  https://developer.apple.com/documentation/speech/speechtranscriber)
- GitHub Changelog: "macos-26 is now generally available for GitHub-hosted runners"
  (2026-02-26) https://github.blog/changelog/2026-02-26-macos-26-is-now-generally-available-for-github-hosted-runners/
- actions/runner-images macos-26 readme
  https://github.com/actions/runner-images/blob/main/images/macos/macos-26-arm64-Readme.md
