# Title

SwiftPM package scaffold: targets, module boundaries, Makefile, formatting

## Summary

Create the complete SwiftPM skeleton for Handsfree: four production targets,
three test targets, resource wiring, a `Makefile` with the canonical developer
commands, and formatting/lint configuration. After this issue, `swift build`,
`swift test`, and `swift format lint --strict` all pass on a clean checkout.

## Context

Handsfree is a pure SwiftPM project by decision — no `.xcodeproj`, buildable
with Command Line Tools only (ADR-006). The module graph and its dependency
rules are fixed in DESIGN.md §3.1/§3.3 and later issues rely on these exact
target names and directory paths.

## Scope

- `Package.swift`, directory skeleton, placeholder source files so every
  target compiles, `Makefile`, `.gitignore`, `VERSION` file, LICENSE, format config.
- No feature logic, no UI, no CI (issue 02), no `.app` bundling (issue 05).

## Detailed Requirements

1. `Package.swift` (swift-tools-version that ships with Swift 6.3):
   - `platforms: [.macOS(.v26)]` (ADR-007). If `.v26` is not a valid SwiftPM
     enum case on the toolchain, use the documented equivalent
     (`.macOS("26.0")`) — do not lower the target.
   - Products/targets:
     - `HandsfreeCore` (library) — no dependencies.
     - `HandsfreeSpeech` (library) — depends on `HandsfreeCore`.
     - `HandsfreeAgent` (library) — depends on `HandsfreeCore`.
     - `HandsfreeApp` (executable) — depends on all three.
     - Test targets: `HandsfreeCoreTests`, `HandsfreeSpeechTests`,
       `HandsfreeAgentTests` (swift-testing style, `import Testing`).
   - **Zero third-party dependencies** (ADR-010). The `dependencies:` array
     stays empty.
   - Resources: declare `Resources/phrases`, `Resources/response-schema.json`,
     `Resources/earcons`, `Resources/scaffolds` as `.copy`/`.process` resources
     of the owning targets (`HandsfreeCore` owns phrases + scaffolds,
     `HandsfreeAgent` owns response-schema.json, `HandsfreeSpeech` owns
     earcons). Create placeholder files so the build is warning-free.
2. Directory skeleton exactly as DESIGN §3.3 (create with placeholder Swift
   files, e.g. `enum HandsfreeCoreModule {}` markers, one per subdirectory).
3. Swift 6 strict concurrency: build settings enable the default Swift 6
   language mode; all placeholder code must compile without concurrency
   warnings.
4. `Makefile` with targets (each currently implemented or stubbed with a clear
   TODO failure message pointing at the owning issue):
   `build` (`swift build`), `test` (`swift test`), `lint`
   (`swift format lint --strict --recursive Sources Tests`), `format`,
   `app` (stub → issue 05), `sign`/`notarize`/`release` (stubs → issue 39),
   `clean`.
5. `.gitignore`: `.build/`, `*.app`, `dist/`, `.DS_Store`, `*.xcodeproj`.
6. `VERSION` file containing `0.1.0-dev` (single source of version truth;
   consumed by issues 05/39).
7. `LICENSE`: MIT, copyright the repository owner. Marked as
   provisional in the file header comment of README? No — LICENSE stays clean;
   the branding gate (issue 41) confirms the final license choice before the
   first public release.
8. Constants file `Sources/HandsfreeCore/AppIdentity.swift`:
   `public enum AppIdentity { public static let bundleID = "io.github.saber5656.handsfree";
   public static let productName = "Handsfree"; public static let logSubsystem = bundleID }`
   — every later issue must reference these, never literal strings (branding
   gate, issue 41, relies on this).

## Acceptance Criteria

- [ ] `swift build` and `swift test` succeed on macOS 26 with CLT only (no Xcode).
- [ ] `swift format lint --strict --recursive Sources Tests` passes.
- [ ] Target dependency graph matches DESIGN §3.1 exactly (verified by
      `swift package describe --type json` in the validation step).
- [ ] No third-party dependencies in `Package.swift`.
- [ ] All directories of DESIGN §3.3 exist and are non-empty.
- [ ] `make build test lint` work; stubbed targets fail with pointer messages.

## Validation

- Run: `swift build && swift test && make lint`.
- Run: `swift package describe --type json | python3 -c '<snippet in PR>'`
  asserting the target→dependency edges (App→Core/Speech/Agent,
  Speech→Core, Agent→Core, Core→∅).
- Fresh-clone check: `git clean -xdf && swift build` (documented in PR).

## Dependencies

None (first issue).

## Non-goals

CI, bundling, icons, any runtime behavior, README content (issue 40).

## Design References

DESIGN.md §3.1, §3.3; ADR-006, ADR-007, ADR-010; ISSUE_PLAN.md wave 0.
