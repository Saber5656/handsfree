# Title

Pre-release branding gate: name decision, identifier freeze, publication scan

## Summary

Execute the pre-release branding checklist: the human maintainer confirms or
changes the product name and license, identifiers are frozen and
consistency-tested, namespace collisions re-verified, and the repository
publication-safety scan is run — the last gate before the repo/releases go
public.

## Context

Naming research (2026-07-08) found the npm/GitHub "handsfree" collision
(archived hand-tracking library) with Homebrew free; the decision was
deferred here. Repository policy requires a history-wide secret/PII scan
before publication. `AppIdentity` (01) keeps a rename bounded. This gate
runs LAST: it verifies artifacts owned by 01/03/05/35/39 and touches docs
owned by 40.

## Scope

- Decision memo + verification + consistency test + scan + repo metadata.
  Mechanical rename sweep only IF the maintainer chooses rename.

## Detailed Requirements

1. Collision re-verification at gate time (evidence table in the PR):
   `brew search <name>` + cask API check, GitHub repo/org search, npm
   (informational), obvious trademark hits via web search (evidence, not
   legal advice), domain availability (informational).
2. Decision memo in the PR: keep `Handsfree` vs ≤ 3 alternatives with
   pros/cons (discoverability, collision, ja pronunciation). **The name and
   license decisions are made by the human maintainer — this issue blocks
   on their explicit sign-off comment.** License: confirm MIT (placed in
   01) or replace; SPDX policy recorded in CONTRIBUTING (recommend: root
   LICENSE only, no per-file headers).
3. `AppIdentityConsistencyTests`
   (`Tests/HandsfreeCoreTests/AppIdentityConsistencyTests.swift`,
   implemented regardless of keep/rename) — assertion table:
   | Artifact | Expected |
   |---|---|
   | rendered Info.plist (via `make app SIGN=none` in a fixture step, or direct template substitution in-test) | `CFBundleIdentifier == AppIdentity.bundleID` |
   | `scripts/make-app.sh` BUNDLE_ID variable | equals `AppIdentity.bundleID` (grep) |
   | `Log` subsystem constant | equals `AppIdentity.logSubsystem` |
   | `SupportPaths` directory name | equals `AppIdentity.productName` |
   | notification category | remains literally `HANDSFREE_TASK` (35's contract; renamed only if 35 is amended) |
   | release artifact name pattern in `release.sh` | contains `AppIdentity.productName` |
4. On KEEP: freeze `io.github.saber5656.handsfree`; close DESIGN §16 #8.
   On RENAME: mechanical sweep — `AppIdentity` constants, UI strings in
   `Sources/HandsfreeApp/**/*.lproj/Localizable.strings` and the phrase
   JSONs' user-visible product mentions (explicit path list; `.xcstrings`
   is forbidden by ADR-006 and must not appear), docs headers
   (README/DESIGN/ISSUE_PLAN), repo rename + redirect check, support-dir
   migration decision recorded (pre-1.0: no migration code; release-noted
   breaking change).
5. Publication-safety scan (policy for going public; evidence pasted):
   - `gitleaks detect --source . --log-opts="--all"` over full history
     (`gitleaks version` recorded; installed ad hoc, not a repo dep);
   - history grep with the exact pattern set: the maintainer's personal
     email(s) other than the intended committer identity, `/Users/<name>`
     absolute home paths, `icloud.com|gmail.com` addresses outside
     LICENSE/committer fields, plus the issue-04 redactor token patterns —
     command lines included in the issue for mechanical execution;
   - contamination found → STOP and escalate to the maintainer with the
     policy options (fresh-repo migration vs history rewrite); do not
     proceed to publication.
6. Repo metadata (exact commands, run after 40's SECURITY.md exists):
   `gh repo edit <owner>/<repo> --description "Drive coding agents
   end-to-end with your voice on macOS" --add-topic macos --add-topic voice
   --add-topic speech --add-topic coding-agent --add-topic codex --add-topic
   accessibility`; enable private vulnerability reporting (gh API or web —
   documented step with verification screenshot); Releases enabled; social
   preview marked maintainer-manual.

## Acceptance Criteria

- [ ] Collision evidence table dated at gate time.
- [ ] Maintainer decision recorded (name + license) — explicit human
      sign-off comment linked.
- [ ] `AppIdentityConsistencyTests` implemented and green (all table rows).
- [ ] Publication scan executed; clean result evidence, or documented
      escalation resolved before close.
- [ ] Repo metadata set + private vulnerability reporting enabled
      (verification evidence); DESIGN §16 #8 closed.

## Validation

Evidence artifacts in the PR (search outputs, gitleaks summary, consistency
test run, `gh repo view` output); maintainer approval comment.

## Dependencies

35, 39, 40 (artifact owners verified by the consistency test; docs edited
here are 40's outputs). Must complete before the first public release.

## Non-goals

Logo/icon design, trademark registration, marketing site, Homebrew cask
submission (v2).

## Design References

docs/research/2026-07-08-naming-collision.md; DESIGN.md §16 (#8); ADR-006;
repository publication-safety policy; issues 01/35/39/40 artifacts.
