# Title

ProjectRegistry: validated project entries and voice name resolution

## Summary

Implement `HandsfreeCore/Projects/ProjectRegistry`: CRUD over the config's
project entries with strict path validation, plus voice-driven project
resolution (exact → alias → bounded fuzzy → disambiguation).

## Context

Projects are the only paths agent turns may write to; validation here is a
security control (DESIGN §7.1, §9.5). Voice resolution feeds `switch_project`
and session-start context (§5.2).

## Scope

- Registry logic + resolution + tests. UI CRUD is issue 33; matching
  primitives come from 19.

## Detailed Requirements

1. API (actor, backed by ConfigStore 03):
   ```swift
   func all() async -> [ProjectEntry]
   func add(name: String, aliases: [String], path: URL) async throws -> ProjectEntry
   func update(_ e: ProjectEntry) async throws
   func remove(id: UUID) async throws
   func defaultProject() async -> ProjectEntry?
   func setDefault(id: UUID?) async throws
   func resolve(spokenName: String, locale: SpeechLocale) async -> ProjectResolution
   func validateForDispatch(_ e: ProjectEntry) async -> [ProjectProblem]
   ```
2. Validation (add/update AND again at every dispatch —
   `validateForDispatch`): path is absolute; exists; is a directory; is a git
   repository (`.git` exists as dir OR file — worktree support); is NOT inside
   `~/Library/Application Support/Handsfree` (§7.1); is not the filesystem
   root or the home directory itself (guard rails). Each failure → typed
   `ProjectProblem` with a spoken-error template key.
3. Name rules: 1–64 chars post-trim; names and aliases unique across the
   registry after normalization (19's normalizer); adding a duplicate →
   typed error.
4. `resolve` pipeline: normalize spoken name → exact name match → exact alias
   match → fuzzy over names+aliases (edit distance ≤ 2, candidates ranked) →
   `ProjectResolution`:
   `.match(entry) | .ambiguous(top: [entry] /* max 2 */) | .none`.
   Ambiguity exists when ≥ 2 candidates tie within distance 1 of each other.
   The FSM speaks a disambiguation prompt from `.ambiguous` (28).
5. ASCII project names spoken inside Japanese utterances must resolve
   (test: 「プロジェクト ハンズフリー」alias + "handsfree" both → same entry;
   STT variants like "hands free" (split) covered by alias guidance — the
   add-flow doc comment tells the Settings UI (33) to suggest space-split
   aliases automatically).
6. Deletion of the default project clears `default_project`; resolution with
   an empty registry → `.none` with a distinct problem (onboarding pointer).

## Acceptance Criteria

- [ ] All validation rules individually tested (incl. worktree `.git` file,
      support-dir rejection, home-dir rejection).
- [ ] Resolution: exact/alias/fuzzy/ambiguous/none — ≥ 12 cases per locale
      including code-switched names.
- [ ] Uniqueness enforcement after normalization (「Handsfree」 vs
      "handsfree" collide).
- [ ] Dispatch-time re-validation catches a path deleted after registration.
- [ ] Config round-trip: entries persist and reload via ConfigStore.

## Validation

`swift test --filter ProjectRegistryTests`.

## Dependencies

03, 19.

## Non-goals

Voice CRUD (v1 registers via Settings only), per-project prompt customization,
`--add-dir` multi-root tasks.

## Design References

DESIGN.md §7.1, §5.2, §9.5; ADR-002 (git-repo requirement).
