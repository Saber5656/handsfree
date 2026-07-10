# Title

`.app` bundle assembly: make-app.sh, Info.plist template, entitlements, icon

## Summary

Implement `scripts/make-app.sh` (invoked by `make app`) turning the SwiftPM
release build into a launchable `dist/Handsfree.app` with correct Info.plist,
resource bundles, icon, and (by default) an ad-hoc signature — CLT-only.

## Context

No Xcode/actool on the reference machine (ADR-006). TCC prompts attribute to
a bundle, so all mic-dependent manual testing needs this issue. Per DESIGN
§10: `make app` ad-hoc signs by default for stable local TCC identity;
`make app SIGN=none` (CI) skips signing; Developer ID signing is exclusively
issue 39.

## Scope

- `scripts/make-app.sh`, `Resources/Info.plist.template`,
  `Resources/Handsfree.entitlements`, `Resources/icon/Handsfree.iconset`
  placeholder PNGs, `make app` target. Not: notarization, distribution zips
  (39).

## Detailed Requirements

1. `Resources/Info.plist.template` (XML plist) with `__VERSION__` /
   `__BUNDLE_ID__` placeholders: `CFBundleIdentifier`, `CFBundleName=Handsfree`,
   `CFBundleExecutable=Handsfree`, `CFBundlePackageType=APPL`,
   `CFBundleShortVersionString`/`CFBundleVersion=__VERSION__`,
   `LSMinimumSystemVersion=26.0`, **`LSUIElement=true`**,
   `NSMicrophoneUsageDescription` ("Handsfree uses the microphone only during
   voice sessions you start, to hear your instructions."),
   `NSSpeechRecognitionUsageDescription` ("Handsfree transcribes your voice
   on-device to drive coding agents."), `CFBundleIconFile=Handsfree`.
2. `Resources/Handsfree.entitlements`: exactly one entitlement,
   `com.apple.security.device.audio-input=true` (DESIGN §9.3). Consumed by
   issue 39; kept here so the bundle inputs live together.
3. `scripts/make-app.sh` (bash, `set -euo pipefail`):
   - `BIN_DIR=$(swift build -c release --product HandsfreeApp --show-bin-path)`
     — never hard-code `.build/release`.
   - Assemble `dist/Handsfree.app/Contents/{MacOS,Resources}`; copy
     `"$BIN_DIR/HandsfreeApp"` → `Contents/MacOS/Handsfree`.
   - Copy every `"$BIN_DIR"/Handsfree_*.bundle` into `Contents/Resources/`
     (SwiftPM names resource bundles `<Package>_<Target>.bundle`); fail
     loudly if `Handsfree_HandsfreeCore.bundle` is absent.
   - Render Info.plist: substitute `__VERSION__` from `VERSION` and
     `__BUNDLE_ID__="io.github.saber5656.handsfree"` (single variable at the
     top of the script). Consistency guard: `grep -q "$BUNDLE_ID"
     Sources/HandsfreeCore/AppIdentity.swift` or fail. After rendering,
     `plutil -lint` and `! grep -q "__" <rendered>` must pass.
   - Icon: `iconutil -c icns Resources/icon/Handsfree.iconset -o
     Contents/Resources/Handsfree.icns`. Commit placeholder iconset PNGs
     (16/32/128/256/512 + @2x; plain glyph, generated once and committed —
     no asset catalog).
   - Signing: default `codesign --force --deep -s - dist/Handsfree.app`
     (ad-hoc, dev-only; no `--options runtime`, no entitlements — TCC
     prompts work with plain ad-hoc; issue 39 owns hardened-runtime signing
     inside-out). `SIGN=none make app` skips this step entirely.
   - On success print the bundle path and, when signed, `codesign -dv`
     summary.
4. `make app` invokes the script honoring `SIGN`; `make clean` removes
   `dist/`.
5. Idempotent re-runs; clear failure messages for: missing release binary,
   missing iconutil, unresolved placeholders, missing Core resource bundle.

## Acceptance Criteria

- [ ] `make app` on a clean checkout produces `dist/Handsfree.app`; re-run
      succeeds.
- [ ] `codesign --verify --strict dist/Handsfree.app` passes (default mode);
      `SIGN=none make app` produces an unsigned bundle and skips verify.
- [ ] `plutil -lint` passes; no `__` placeholders remain.
- [ ] `open dist/Handsfree.app` launches, process visible in Activity
      Monitor, **no Dock icon** (the issue-01 placeholder keeps a run loop).
- [ ] Bundle smoke: `HANDSFREE_SMOKE=1 dist/Handsfree.app/Contents/MacOS/Handsfree`
      prints the issue-01 `smoke-ok` line (proving `Bundle.module` resolution
      inside the assembled bundle) and exits 0.
- [ ] `shellcheck scripts/make-app.sh` clean (tool run locally; output in PR).

## Validation

Manual run transcript in the PR: `make app`, `codesign --verify --strict`,
`plutil -lint`, smoke-hook output, plus a note confirming no Dock icon.

## Dependencies

01.

## Non-goals

Developer ID signing/notarization/zips (39), real icon design, FakeCodex
bundling (added by issue 34's amendment).

## Design References

DESIGN.md §3.2, §3.3, §9.3, §10; ADR-006; research doc (toolchain table).
