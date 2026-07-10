# Title

SwiftPM package scaffold: targets, module boundaries, Makefile, formatting

## Summary

Create the complete SwiftPM skeleton for Handsfree: four production targets,
a test-support target, three test targets, target-relative resource wiring, a
`Makefile` with the canonical developer commands, and formatting checks. After
this issue, `swift build`, `swift test`, and `swift format lint --strict
--recursive Sources Tests` all pass on a clean checkout.

## Context

Handsfree is a pure SwiftPM project by decision — no `.xcodeproj`, buildable
with Command Line Tools only (ADR-006). The module graph and directory paths
are fixed in DESIGN.md §3.1/§3.3 and later issues rely on these exact names.

## Scope

- `Package.swift`, source/test directory skeleton, placeholder sources so
  every target compiles and the app placeholder stays alive, `Makefile`,
  `.gitignore`, `VERSION`, LICENSE.
- NOT in scope: `.github/` (issue 02), `scripts/` and top-level `Resources/`
  bundle inputs (issue 05), FakeCodex sources (issue 17), any feature logic.

## Detailed Requirements

1. `Package.swift` (swift-tools-version shipped with Swift 6.3), package name
   `Handsfree`:
   - `platforms: [.macOS(.v26)]` (ADR-007). If `.v26` is not a valid case on
     this toolchain, use `.macOS("26.0")` — do not lower the target.
   - Targets and dependency edges (DESIGN §3.1 — Core consumes the domain
     modules' protocol surfaces):
     - `HandsfreeSpeech` (library) — no target dependencies.
     - `HandsfreeAgent` (library) — no target dependencies.
     - `HandsfreeCore` (library) — depends on `HandsfreeSpeech`, `HandsfreeAgent`.
     - `HandsfreeApp` (executable) — depends on all three.
     - `HandsfreeTestSupport` (library) — depends on `HandsfreeSpeech`,
       `HandsfreeAgent`, `HandsfreeCore`; may ONLY be depended on by test
       targets (never by production targets).
     - Test targets `HandsfreeCoreTests` / `HandsfreeSpeechTests` /
       `HandsfreeAgentTests` — each depends on its module + `HandsfreeTestSupport`
       (swift-testing, `import Testing`).
   - **Zero third-party dependencies** (ADR-010): package `dependencies: []`.
   - Target-relative resources (create placeholder files so the build is
     warning-free): `Sources/HandsfreeCore/Resources/phrases/.gitkeep` and
     `…/scaffolds/.gitkeep`, `Sources/HandsfreeAgent/Resources/response-schema.json`
     (placeholder `{}`), `Sources/HandsfreeSpeech/Resources/earcons/.gitkeep`
     — declared with `.copy` on each owning target.
2. Directory skeleton: create exactly `Sources/{HandsfreeApp,HandsfreeCore,
   HandsfreeSpeech,HandsfreeAgent,HandsfreeTestSupport}` with the DESIGN §3.3
   subdirectories under each (marker file per subdirectory, e.g.
   `enum SessionFSMModule {}`), plus `Tests/{HandsfreeCoreTests,
   HandsfreeSpeechTests,HandsfreeAgentTests,Fixtures}`. Do NOT create
   `.github/`, `scripts/`, top-level `Resources/`, or `Sources/FakeCodex`
   (owned by 02/05/17).
3. Swift 6 language mode; all placeholder code compiles without concurrency
   warnings.
4. Shared scaffold types (exact contents in this issue's PR):
   - `Sources/HandsfreeCore/AppIdentity.swift`:
     `public enum AppIdentity { public static let bundleID = "io.github.saber5656.handsfree";
     public static let productName = "Handsfree"; public static let logSubsystem = bundleID }`.
     Later issues must reference these, never literal strings (issue 41 relies
     on this).
   - `Sources/HandsfreeCore/SpeechLocale.swift`:
     `public enum SpeechLocale: String, Sendable, CaseIterable, Codable {
     case jaJP = "ja-JP"; case enUS = "en-US" }` — the app-wide spoken-locale
     type consumed by issues 19/21/23/24/28.
   - `Sources/HandsfreeTestSupport/TestClock.swift`: a hand-rolled test clock
     (~50 lines, conforming to `Clock<Duration>` semantics: `now`,
     `advance(by:)`, `sleep(until:)` resuming pending sleepers) used by all
     timeout tests. Include one example test proving advance/sleep semantics.
   - `Sources/HandsfreeApp/main` placeholder: a minimal AppKit entry point
     (`NSApplication.shared`, `app.run()`) so the process stays alive when
     launched, plus a smoke hook: when env `HANDSFREE_SMOKE=1`, print
     `smoke-ok bundle=<AppIdentity.bundleID>` and the count of files found in
     the Core phrases resource directory via `Bundle.module`, then exit 0
     (consumed by issue 05's bundle verification).
5. `Makefile` targets: `build`, `test`, `lint`
   (`swift format lint --strict --recursive Sources Tests`), `format`,
   `clean`, and stubs `app`/`sign`/`notarize`/`release` that exit 1 with
   `"implemented by issue NN"` messages. **No `.swift-format` config file is
   created**: code must pass the toolchain's default `swift format` settings
   (avoids config drift; revisit only via a dedicated PR).
6. `.gitignore`: `.build/`, `dist/`, `*.app`, `.DS_Store`, `*.xcodeproj`.
7. `VERSION` file containing `0.1.0-dev` (single source of version truth for
   issues 05/39).
8. `LICENSE`: MIT, copyright the repository owner. The branding gate (issue
   41) confirms the final license before the first public release.

## Acceptance Criteria

- [ ] `swift build` and `swift test` succeed on macOS 26 with CLT only.
- [ ] `swift format lint --strict --recursive Sources Tests` passes with no
      config file present.
- [ ] `swift package describe --type json` shows exactly these edges:
      Core→{Speech, Agent}; App→{Core, Speech, Agent}; Speech→{}; Agent→{};
      TestSupport→{Core, Speech, Agent}; each test target→{its module,
      TestSupport}; and no production target depends on TestSupport
      (assertion script included in the PR and pasted output).
- [ ] `dependencies: []` in `Package.swift` (zero third-party).
- [ ] `HANDSFREE_SMOKE=1 swift run HandsfreeApp` prints the `smoke-ok` line
      and exits 0.
- [ ] TestClock example test passes.

## Validation

- `swift build && swift test && make lint`.
- The `swift package describe` assertion script run (output in PR).
- Fresh-clone check: `git clean -xdf && swift build` (documented in PR).

## Dependencies

None (first issue).

## Non-goals

CI, bundling/scripts, icons, FakeCodex, any runtime behavior beyond the smoke
hook, README content (issue 40).

## Design References

DESIGN.md §3.1, §3.3; ADR-006, ADR-007, ADR-010; ISSUE_PLAN.md wave 0.
