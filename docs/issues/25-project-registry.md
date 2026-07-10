# Title

ProjectRegistry: validated project entries and voice name resolution

## Summary

Implement `HandsfreeCore/Projects/ProjectRegistry`: CRUD over the config's
project entries with strict path validation, plus deterministic voice-driven
project resolution.

## Context

Projects are the only paths agent turns may write to (T4). Issue 03 already
validates shape/existence/ownership at config load; the registry re-validates
at add/update AND at every dispatch, and adds the git/support-dir rules. The
error template keys used here are defined in issue 19's dictionary set.

## Scope

- Registry actor + resolution + typed problems. UI CRUD is 33; matching
  primitives from 19.

## Detailed Requirements

1. API (actor, backed by ConfigStore 03):
   ```swift
   func all() async -> [ProjectEntry]
   func add(name: String, aliases: [String], path: URL,
            defaultModel: String? = nil) async throws -> ProjectEntry
   func update(_ e: ProjectEntry) async throws
   func remove(id: UUID) async throws
   func defaultProject() async -> ProjectEntry?
   func setDefault(id: UUID?) async throws
   func resolve(spokenName: String, locale: SpeechLocale) async -> ProjectResolution
   func validateForDispatch(_ e: ProjectEntry) async -> [ProjectProblem]
   ```
2. Validation (add/update AND `validateForDispatch`; each failure a typed
   case): path absolute (`.notAbsolute`), exists (`.missing`), directory
   (`.notADirectory`), owned by current uid (`.notOwned`), is a git repo —
   `.git` exists as dir or file, covering worktrees (`.notAGitRepo`), not
   inside the Handsfree support directory (`.insideSupportDir`), not `/` or
   the home directory itself (`.guardRailPath`).
   Template-key mapping: `.missing/.notAbsolute/.notADirectory/.notOwned/
   .guardRailPath` → `error.project_invalid_path`; `.notAGitRepo` →
   `error.project_not_git`; `.insideSupportDir` → `error.project_invalid_path`.
3. Name rules: trim → 1…64 chars; aliases ≤ 8, each ≤ 64; names+aliases
   unique across the registry after issue-19 normalization (「Handsfree」vs
   "handsfree" collide). Violations throw
   `ProjectRegistryError.duplicateName(existing: UUID)` /
   `.invalidName(reason)` / `.tooManyAliases`.
4. `resolve` — deterministic algorithm:
   1. normalize the spoken name (19's normalizer);
   2. exact name match → `.match`;
   3. exact alias match → `.match`;
   4. fuzzy: compute the BEST edit distance per entry across its
      name+aliases (≤ 2 to qualify); rank by (distance, then registry
      insertion order as the stable tiebreak);
   5. result: one qualifier → `.match`; ≥ 2 qualifiers whose top-two
      distances differ by ≤ 1 → `.ambiguous(top: [first, second])`;
      otherwise best wins → `.match`; none →
      `.none(reason: .noMatch)` — with an EMPTY registry always
      `.none(reason: .emptyRegistry)` (template `error.registry_empty`).
   ```swift
   enum ProjectResolution { case match(ProjectEntry)
       case ambiguous(top: [ProjectEntry])           // exactly 2
       case none(reason: NoneReason) }               // .noMatch | .emptyRegistry
   ```
5. Alias guidance: doc comment instructing the Settings UI (33) to suggest
   space-split alias variants for ASCII names (STT often splits
   "handsfree" → "hands free"); resolution itself also normalizes internal
   whitespace so "hands free" matches "handsfree" (test).
6. Removing the default project clears `projects.default_project`.

## Acceptance Criteria

- [ ] Every validation rule individually tested (worktree `.git` file case,
      support-dir, home dir, non-owned via injected ownership checker).
- [ ] Resolution: exact / alias / fuzzy-single / ambiguous-top2 /
      none(noMatch) / none(emptyRegistry) — ≥ 14 cases per locale incl.
      code-switched names and whitespace-split variants.
- [ ] Determinism: same inputs → same resolution (order-stability test with
      3 similarly-named entries).
- [ ] Uniqueness-after-normalization enforcement.
- [ ] Dispatch-time re-validation catches a path deleted after registration.
- [ ] Config round-trip via ConfigStore.

## Validation

`swift test --filter ProjectRegistryTests`.

## Dependencies

03, 19.

## Non-goals

Voice CRUD (v1 registers via Settings only), per-project prompt
customization, `--add-dir` multi-root tasks.

## Design References

DESIGN.md §7.1, §5.2, §9.5, §9.2 (T4); ADR-002 (git-repo requirement);
issue 19 (template keys).
