# Title

CI workflow: build, test, lint on macos-26 with SHA-pinned actions

## Summary

Add `.github/workflows/ci.yml` running build + tests + lint + an unsigned
`.app` assembly smoke on every PR and push to `main`, on GitHub-hosted
`macos-26` runners, and establish the repository's live-test naming
convention.

## Context

macos-26 arm64 runners are GA since 2026-02-26 (research doc). CI must exclude
tests that need a microphone, downloaded speech assets, or a logged-in codex
CLI (DESIGN §11). Supply-chain rules require SHA-pinned actions (DESIGN §9.2
T8). DESIGN §10 requires the CI to include the `make app` smoke, so this
issue depends on issue 05.

## Scope

- `ci.yml`, `Tests/README.md` (test-convention doc), one permanent
  live-convention fixture test. Release workflow is issue 39.

## Detailed Requirements

1. **Live-test convention** (used by every later issue; documented in
   `Tests/README.md`): test suites that require hardware, speech assets,
   network, or a real codex login are placed in suites whose type name ends
   in `LiveTests`. CI excludes them with `swift test --skip 'LiveTests$'`;
   locally they run with `swift test --filter 'LiveTests$'` (or a narrower
   filter). Rationale: SwiftPM's `--filter/--skip` take a single regex over
   test identifiers; there is no stable tag-based CLI filter on the pinned
   toolchain.
2. Permanent fixture: `Tests/HandsfreeCoreTests/LiveConventionLiveTests.swift`
   containing one trivially-passing test. It proves the skip mechanism in CI
   logs (skipped there, runs locally) and stays in the repo as the
   convention's canary.
3. Triggers: `pull_request` (all branches), `push` to `main`. Concurrency
   group `ci-${{ github.ref }}`, `cancel-in-progress: true`.
4. Single job `build-test`, `runs-on: macos-26`, `timeout-minutes: 30`,
   workflow-level `permissions: contents: read`, no secrets:
   - `actions/checkout` and `actions/cache` pinned to full 40-char commit
     SHAs with version comments.
   - Print toolchain: `sw_vers && swift --version`.
   - Cache `.build` keyed on hash of `Package.swift` + `Sources/**`.
   - `swift build -c debug`.
   - `swift test --skip 'LiveTests$'` (as issues land, phrase/schema/contract
     validation tests run here automatically — no separate step needed).
   - `make lint`.
   - `make app SIGN=none` (unsigned assembly smoke per DESIGN §10) followed by
     `test -x dist/Handsfree.app/Contents/MacOS/Handsfree`.

## Acceptance Criteria

- [ ] CI green on the PR introducing it (run linked in the PR).
- [ ] Every `uses:` is pinned to a 40-char SHA with a version comment.
- [ ] `permissions: contents: read` at workflow level; no secret references.
- [ ] CI log shows `LiveConventionLiveTests` skipped; the same suite passes
      locally with `swift test --filter 'LiveTests$'` (both shown in PR).
- [ ] `make app SIGN=none` step passes in CI.

## Validation

- Scratch commits demonstrating (a) failing unit test → red, (b) revert →
  green; both runs linked in the PR.
- `actionlint` run locally (not a repo dependency); output pasted in the PR.

## Dependencies

01, 05.

## Non-goals

Release/signing workflow (39), coverage upload, matrix builds, README badge
(issue 40 owns README content).

## Design References

DESIGN.md §10, §11, §9.2 (T8); ISSUE_PLAN.md §6; research doc (runner GA).
