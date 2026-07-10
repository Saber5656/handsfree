# Title

`.app` bundle assembly: make-app.sh, Info.plist template, entitlements, icon

## Summary

Implement `scripts/make-app.sh` (invoked by `make app`) that turns the SwiftPM
release build into a launchable `dist/Handsfree.app` with correct Info.plist,
resources, icon, and an ad-hoc code signature — using CLT-only tools.

## Context

No Xcode/actool exists on the reference machine (ADR-006, research doc).
TCC permission prompts attribute to a bundle, so all mic-dependent manual
testing from wave 1 onward needs this issue. Formal Developer ID signing is
issue 39; ad-hoc signing here keeps local TCC identity stable.

## Scope

- `scripts/make-app.sh`, `Resources/Info.plist.template`,
  `Resources/Handsfree.entitlements`, `Resources/icon/` placeholder iconset,
  `make app` target. Not: notarization, distribution zips (39).

## Detailed Requirements

1. `Resources/Info.plist.template` (XML plist) with placeholders
   `__VERSION__`, `__BUNDLE_ID__`:
   - `CFBundleIdentifier=__BUNDLE_ID__`, `CFBundleName=Handsfree`,
     `CFBundleExecutable=Handsfree`, `CFBundlePackageType=APPL`,
     `CFBundleShortVersionString=__VERSION__`, `CFBundleVersion=__VERSION__`,
     `LSMinimumSystemVersion=26.0`, **`LSUIElement=true`**,
     `NSMicrophoneUsageDescription` (exact string: "Handsfree uses the
     microphone only during voice sessions you start, to hear your
     instructions."), `NSSpeechRecognitionUsageDescription` ("Handsfree
     transcribes your voice on-device to drive coding agents."),
     `CFBundleIconFile=Handsfree`.
2. `Resources/Handsfree.entitlements`: exactly one entitlement,
   `com.apple.security.device.audio-input=true` (DESIGN §9.3).
3. `scripts/make-app.sh` (bash, `set -euo pipefail`):
   - `swift build -c release --product HandsfreeApp`.
   - Assemble `dist/Handsfree.app/Contents/{MacOS,Resources}`.
   - Copy executable → `Contents/MacOS/Handsfree`.
   - Copy **all SwiftPM resource bundles** (`.build/release/*.bundle`) into
     `Contents/Resources/` — required for `Bundle.module` resolution at
     runtime; verify at least the Core bundle exists, else fail loudly.
   - Render Info.plist: substitute `__VERSION__` from the `VERSION` file and
     `__BUNDLE_ID__` = `io.github.saber5656.handsfree` (single definition at
     the top of the script; must match `AppIdentity.bundleID` — add a grep
     assertion comparing the two).
   - Icon: `iconutil -c icns Resources/icon/Handsfree.iconset -o
     Contents/Resources/Handsfree.icns`. Commit a placeholder iconset
     (16/32/128/256/512 + @2x PNGs — plain glyph on transparent background;
     generate once with any tool and commit the PNGs; no asset catalog).
   - Ad-hoc sign: `codesign --force --deep -s - --entitlements
     Resources/Handsfree.entitlements dist/Handsfree.app` (note: `--deep` is
     acceptable for ad-hoc dev builds only; issue 39 signs nested items
     explicitly).
   - Print the bundle path and `codesign -dv` summary on success.
4. `make app` invokes the script; `make clean` removes `dist/`.
5. Script must be idempotent (safe re-run) and fail with clear messages when:
   release binary missing, iconutil missing, template placeholders unresolved
   (grep for `__` in the rendered plist).

## Acceptance Criteria

- [ ] `make app` on a clean checkout produces `dist/Handsfree.app`.
- [ ] `codesign --verify --strict dist/Handsfree.app` passes (ad-hoc).
- [ ] `plutil -lint` passes on the rendered Info.plist; no `__` placeholders remain.
- [ ] `open dist/Handsfree.app` launches without crash (placeholder app from
      issue 01 — a bare process is fine), appears in Activity Monitor, no Dock
      icon (LSUIElement).
- [ ] Resource bundles present under `Contents/Resources/` and loadable
      (`Bundle.module` smoke test executed by the app placeholder at launch,
      logging success).

## Validation

Manual run transcript in the PR: `make app && codesign --verify --strict … &&
plutil -lint …` plus a screenshot/note that no Dock icon appears. Shell check:
`shellcheck scripts/make-app.sh` output pasted (installed locally; not a repo
dependency).

## Dependencies

01.

## Non-goals

Developer ID signing, notarization, DMG/zip packaging, real icon design.

## Design References

DESIGN.md §3.2, §9.3, §10; ADR-006; research doc (toolchain table).
