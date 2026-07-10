# Title

CI workflow: build, test, lint on macos-26 with SHA-pinned actions

## Summary

Add `.github/workflows/ci.yml` running build + tests + lint + resource-schema
checks on every PR and push to `main`, on GitHub-hosted `macos-26` runners.

## Context

macos-26 arm64 runners are GA since 2026-02-26 (research doc
2026-07-08-macos-speech-and-toolchain.md). CI must exclude tests that need a
microphone, downloaded speech assets, or a logged-in codex CLI; those are
tagged and run manually (DESIGN §11). Supply-chain rules require SHA-pinned
actions (DESIGN §9.2 T8).

## Scope

- One workflow file `ci.yml` + a `Tests/README.md` note about test tags.
- Release workflow is issue 39. No caching optimizations beyond SwiftPM cache.

## Detailed Requirements

1. Triggers: `pull_request` (all branches), `push` to `main`. Concurrency
   group `ci-${{ github.ref }}` with `cancel-in-progress: true`.
2. Single job `build-test`, `runs-on: macos-26`, `timeout-minutes: 30`:
   - `actions/checkout` pinned to a full commit SHA (comment records the tag).
   - Print toolchain: `sw_vers && swift --version`.
   - Cache `.build` keyed on `Package.swift` + sources hash
     (`actions/cache` pinned by SHA).
   - `swift build -c debug`.
   - `swift test --skip-tag live` — tests requiring hardware/network/codex are
     tagged `live` (swift-testing `.tags(.live)`, tag defined in issue 01's
     test support or first use here).
   - `make lint`.
3. `permissions: contents: read` at workflow top level (least privilege).
4. No secrets referenced anywhere in this workflow.
5. Badge: add CI status badge markdown snippet to the PR description for later
   README use (README itself is issue 40).
6. Document in `Tests/README.md`: tag taxonomy (`live` = requires mic, speech
   assets, or real codex; excluded in CI), how to run them locally
   (`swift test --filter-tag live`).

## Acceptance Criteria

- [ ] CI passes on the PR introducing it (green run linked in the PR).
- [ ] Every `uses:` is pinned to a 40-char commit SHA with a version comment.
- [ ] Workflow has `permissions: contents: read` and no secret usage.
- [ ] A deliberately `live`-tagged dummy test is skipped in CI (visible in logs)
      and runs locally.

## Validation

- Push a scratch commit with (a) a failing unit test → CI red; (b) revert →
  green. Link both runs in the PR.
- `actionlint` (run via `brew install actionlint` locally, not added to repo
  deps) reports no errors — paste output into the PR.

## Dependencies

01.

## Non-goals

Release/signing workflow (39), coverage upload, matrix builds, Linux.

## Design References

DESIGN.md §10, §11, §9.2 (T8); ISSUE_PLAN.md §6; research doc (runner GA).
