# Title

Pre-release branding gate: name decision, identifier freeze, publication scan

## Summary

Execute the pre-release branding checklist: confirm or change the product
name, freeze the bundle identifier and license, re-verify namespace
collisions, and run the repository publication-safety scan — the last gate
before the repo/releases go public.

## Context

Naming research (2026-07-08) found the npm/GitHub "handsfree" collision
(Oz Ramos' archived hand-tracking library) with Homebrew free; the decision
was deferred to this gate. Repository policy requires a history-wide
secret/PII scan before any repo is made public. `AppIdentity` (01) and the
bundle scripts (05/39) were built so a rename is a bounded change.

## Scope

- Decision + verification + mechanical rename sweep IF renamed. No feature
  code.

## Detailed Requirements

1. Re-verify collisions at gate time (record evidence in this issue's PR):
   `brew search <name>` / cask API, GitHub repo/org search, npm (informational
   — we ship no npm package), obvious trademark hits (search evidence, not
   legal advice), domain availability (informational).
2. Present the maintainer a decision memo (in-PR): keep `Handsfree` vs top ≤ 3
   alternatives with pros/cons (discoverability, collision, ja pronunciation).
   **The name decision itself is made by the human maintainer** — the issue
   blocks on their sign-off comment.
3. On KEEP: freeze `AppIdentity.bundleID = io.github.saber5656.handsfree`;
   assert consistency across: Info.plist template, make-app.sh, release
   scripts, log subsystem, notification category, `SupportPaths` (a single
   test `AppIdentityConsistencyTests` greps rendered artifacts — implement it
   now either way).
4. On RENAME: mechanical sweep — `AppIdentity` constants, user-visible strings
   (`L10n` catalogs), docs (README/DESIGN/ISSUE_PLAN headers), repo
   rename note + redirect check, support-directory migration decision
   (document: pre-1.0, no migration code — breaking change acceptable and
   release-noted).
5. License confirmation: maintainer confirms MIT (placed in 01) or selects
   another; SPDX header policy decided (none vs per-file — recommend none +
   root LICENSE only; record in CONTRIBUTING).
6. Publication-safety scan (repository policy for going public):
   - `gitleaks detect --source . --log-opts="--all"` (full history) — run
     locally; paste summary (tool installed ad hoc, not a repo dependency);
   - manual grep of history for personal paths/emails beyond the intended
     author identity (`git log --all -p | grep -iE "<patterns in issue>"`
     guidance included);
   - if contamination is found: follow the policy decision tree (scrub via
     fresh repo vs history rewrite) — escalate to the maintainer, do NOT
     proceed to publication.
7. Repo metadata for launch: description, topics
   (`macos, voice, speech, coding-agent, codex, accessibility`), social
   preview placeholder, Releases enabled, private-vulnerability-reporting
   enabled (SECURITY.md dependency from 40).

## Acceptance Criteria

- [ ] Collision evidence table dated at gate time committed in the PR.
- [ ] Maintainer decision recorded (name + license) — explicit human sign-off.
- [ ] `AppIdentityConsistencyTests` implemented and green.
- [ ] Publication scan executed with clean result (or escalation documented
      and resolved before close).
- [ ] Repo metadata set; DESIGN §16 known-unknown #8 closed.

## Validation

Evidence artifacts in the PR (search outputs, scan summary, consistency test
run); maintainer approval comment.

## Dependencies

None hard (parallel with wave 5), but MUST complete before the first public
release/publication of the repository.

## Non-goals

Logo/icon design (tracked separately if desired), trademark registration,
marketing site.

## Design References

docs/research/2026-07-08-naming-collision.md; DESIGN.md §16 (#8); ADR-006
(identifier plumbing); repository publication-safety policy.
