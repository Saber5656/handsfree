# ADR-006: Pure SwiftPM package with script-assembled `.app` (CLT-only pipeline)

- Status: Accepted (2026-07-08)

## Context

The development machine (and the target contributor baseline) has Swift 6.3
Command Line Tools but **no full Xcode**: `xcodebuild` and `actool` are absent;
`codesign`, `notarytool`, and `iconutil` are present (research 2026-07-08).
Implementation is largely delegated to CLI coding agents, which cannot operate
Xcode GUI projects.

## Decision

- No `.xcodeproj`. The repository is a pure SwiftPM package
  (`Package.swift`, four production targets per DESIGN §3.1).
- `Handsfree.app` is assembled by `scripts/make-app.sh`: copy the release
  binary, render `Info.plist` from a template (usage strings, `LSUIElement`,
  version stamp), copy `Resources/`, build `.icns` with `iconutil` from
  pre-rendered PNGs. **No asset catalogs anywhere** (`actool` unavailable).
- Signing and notarization run with CLT tools only:
  `codesign --options runtime` + entitlements, `xcrun notarytool submit --wait`,
  `xcrun stapler staple`.
- UI strings use classic `.lproj/Localizable.strings` (not `.xcstrings`, which
  needs Xcode tooling).

## Rationale

Keeps every build/test/release step executable by a headless CLI agent and by
CI without Xcode-version coupling; removes the largest local-toolchain
prerequisite for contributors.

## Consequences

- App icon and any images are pre-rendered PNG resources committed to the repo.
- SwiftUI previews are unavailable; UI iteration relies on `make app` + launch.
- TCC-dependent testing must use the assembled bundle (bare `swift run`
  attributes permissions to the terminal) — documented in CONTRIBUTING.
- If a future feature genuinely requires Xcode-only tooling, that is a new ADR.
