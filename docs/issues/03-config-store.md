# Title

Config store: versioned schema, atomic writes, strict permissions

## Summary

Implement `HandsfreeCore/Config`: the typed configuration schema (DESIGN
Appendix C.2), a `ConfigStore` actor with atomic load/save, `0600/0700`
permissions, unknown-key preservation, validation/clamping, and a change
stream.

## Context

Every subsystem reads configuration through this store. It is also a security
boundary (DESIGN §9.5): values like `agent.codex_path` and project paths must
be validated here, and the file must never contain secrets (§9.4).

## Scope

- Schema types, store actor, validation, migration hook, tests.
- No UI (issues 32/33), no project-registry voice matching (25).

## Detailed Requirements

1. Location: `~/Library/Application Support/Handsfree/config.json`. The
   directory component uses `AppIdentity.productName`. Support-dir resolution
   lives in `SupportPaths.swift` (shared with transcript store, issue 27) and
   must create directories with permissions `0700`; the file is written `0600`
   (POSIX attributes via FileManager; assert in tests).
2. Schema: implement exactly the keys and defaults of DESIGN Appendix C.2 as
   nested Codable structs with a top-level `version: Int` (current = 1).
   Field-level rules:
   - `general.locale_mode ∈ {auto, ja, en}`; `idle_timeout_sec` clamp 10…300.
   - `general.hotkey`: `{key: String, modifiers: [String]}`, validated against
     the keys/modifiers table published by issue 30 (until then: non-empty
     strings).
   - `voice.speaking_rate` clamp 0.1…1.0; `narration_verbosity ∈ {quiet,
     milestones, verbose}`.
   - `agent.codex_path`: `nil` or an absolute path to an existing regular file
     that is NOT world-writable; invalid → decode succeeds but value is
     replaced by `nil` with a logged warning (never crash on bad config).
   - `agent.max_turn_seconds` clamp 60…7200; `max_concurrent_tasks` clamp 1…10.
   - `privacy.transcript_retention_days` clamp 0…365.
   - `projects.entries[]`: `{id: UUID, name: String(1…64), aliases: [String] ≤8,
     path: String absolute, default_model: String?}` — path validation
     (exists/dir/git) is owned by issue 25; here only shape validation.
3. `ConfigStore` (actor):
   - `load()` at init: missing file → defaults persisted; corrupt JSON → rename
     to `config.json.corrupt-<timestamp>`, load defaults, log error.
   - `update(_ mutate: (inout Config) -> Void) async throws` — serialized
     read-modify-write; save is temp-file + `rename()` atomic replace, fsync'd.
   - **Unknown-key preservation**: keep the raw top-level JSON dictionary from
     disk; on save, deep-merge typed values over it so keys written by newer
     versions survive round-trips (DESIGN §7.2). Implement with
     `JSONSerialization` merge; document the merge rules in doc comments.
   - `changes: AsyncStream<Config>` emitting after each successful save.
   - Migration hook: `migrate(raw: [String: Any], fromVersion: Int)` — v1 is a
     no-op passthrough but the call site and tests must exist.
4. No secret material: doc comment on the schema root states the §9.4 rule and
   PR reviewers' obligation; add a unit test that fails if any schema key path
   contains `token`, `key`, `secret`, `password` (tripwire for future edits).

## Acceptance Criteria

- [ ] Fresh start creates dir `0700`, file `0600`, valid defaults.
- [ ] Round-trip preserves an injected unknown key `{"future_key": {"x":1}}`.
- [ ] Corrupt file → app-usable defaults + `.corrupt-*` sibling; no throw to callers.
- [ ] All clamps/enum validations behave per the table above (test per field group).
- [ ] Concurrent `update` calls from 100 tasks serialize without data loss
      (stress test with a counter field in a test-only extension).
- [ ] Secret-keyname tripwire test present and green.

## Validation

`swift test --filter ConfigStoreTests` green; include a test using a real temp
HOME (`FileManager` temp dir override injected via initializer) — the store
must accept an injectable root for tests.

## Dependencies

01.

## Non-goals

Settings UI, keychain, per-project config files, hotkey semantic validation.

## Design References

DESIGN.md §7.2, Appendix C.2, §9.4, §9.5; ADR-010.
