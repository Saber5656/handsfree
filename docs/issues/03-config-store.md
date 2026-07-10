# Title

Config store: versioned schema, atomic writes, strict permissions

## Summary

Implement `HandsfreeCore/Config`: the typed configuration schema (DESIGN
Appendix C.2), a `ConfigStore` actor with atomic load/save, `0600/0700`
permissions, unknown-key preservation, per-field validation/clamping, and a
change stream.

## Context

Every subsystem reads configuration through this store. It is a security
boundary (DESIGN §9.5): all path fields are validated here, and the file must
never contain secrets (§9.4).

## Scope

- Schema types, store actor, validation, migration hook, tests. No UI
  (32/33), no voice/project resolution (25).

## Detailed Requirements

1. Location: `~/Library/Application Support/Handsfree/config.json`
   (directory name from `AppIdentity.productName`). `SupportPaths.swift`
   (shared with issue 27) resolves the root, accepts an injectable override
   root for tests, and creates directories `0700`; the file is written `0600`
   (POSIX attributes asserted in tests).
2. Schema: exactly the keys and defaults of DESIGN Appendix C.2 (including
   `general.onboarding_completed` and `agent.codex_path_confirmed_path`) as
   nested Codable structs with top-level `version: Int` (current = 1).
   **Validation policy**: JSON-unparseable file ⇒ corrupt-file path (rule 4);
   a parseable file with invalid field values NEVER fails load — each invalid
   field is individually reset to its default with a logged warning.
   Field rules (complete):
   - `general.locale_mode` ∈ {`auto`,`ja`,`en`}; `idle_timeout_sec` clamp
     10…300; `launch_at_login`, `onboarding_completed` booleans;
     `hotkey` = `{key: String, modifiers: [String]}` — shape-validated
     (non-empty key, modifiers ⊆ {control,option,command,shift}, ≥1 modifier);
     semantic key-name validation is upgraded by issue 32 using issue 30's
     `KeyMap` (documented hook point here).
   - `voice.stt_provider` ∈ {`apple`}; `voice.tts_provider` ∈ {`apple`};
     `tts_voice_ja`/`tts_voice_en` nullable non-empty strings ≤ 256;
     `speaking_rate` clamp 0.1…1.0;
     `narration_verbosity` ∈ {`quiet`,`milestones`,`verbose`}.
   - `agent.kind` ∈ {`codex`}; `agent.model` nullable non-empty ≤ 128;
     `max_turn_seconds` clamp 60…7200; `max_concurrent_tasks` clamp 1…10.
   - **Path fields** (`agent.codex_path`, `agent.codex_path_confirmed_path`,
     `projects.entries[].path`): must be absolute, exist, and be owned by the
     current uid; `codex_path*` must additionally be a regular file that is
     not world-writable and whose parent directory is not world-writable;
     project paths must be directories. Violation ⇒ field reset to `nil`
     (for project entries: the entry is kept but flagged `invalid_path=true`
     in memory so issue 25/33 can surface it; it is excluded from dispatch).
     Git-repository and support-dir checks are owned by issue 25.
   - `policy.allow_tier3`, `policy.tier3_screen_confirm` booleans.
   - `privacy.store_transcripts` boolean; `transcript_retention_days` clamp
     0…365.
   - `projects.default_project`: nullable UUID; if it does not reference an
     existing `entries[].id`, reset to `nil` with warning.
   - `projects.entries[]`: `{id: UUID, name: String(1…64),
     aliases: [String] max 8 items each ≤ 64, path, default_model: String?
     ≤ 128}`; duplicate ids ⇒ later duplicates dropped with warning.
3. `ConfigStore` (actor):
   - `load()` at init: missing file → defaults persisted; unparseable JSON →
     rename to `config.json.corrupt-<timestamp>`, load defaults, log error.
   - `update(_ mutate: (inout Config) -> Void) async throws` — serialized
     read-modify-write; save = temp file + `rename()` atomic replace, fsync.
   - Unknown-key preservation: keep the raw top-level JSON from disk; on save
     deep-merge typed values over it so newer-version keys survive
     round-trips (merge rules in doc comments; `JSONSerialization`-based).
   - `changes: AsyncStream<Config>` emitting after each successful save
     (multicast to any number of consumers; document semantics).
   - Migration hook `migrate(raw:fromVersion:)` — v1 no-op, call site + test.
4. Secret tripwire: unit test fails if any schema key path matches
   `(?i)(api_key|apikey|token|secret|password|credential)` — with an explicit
   empty allowlist constant next to the test for future justified exceptions.
   Doc comment on the schema root states the §9.4 no-secrets rule.

## Acceptance Criteria

- [ ] Fresh start creates dir `0700`, file `0600`, valid defaults.
- [ ] Round-trip preserves an injected unknown key `{"future_key":{"x":1}}`.
- [ ] Unparseable file → defaults + `.corrupt-*` sibling; parseable file with
      per-field garbage → every field group individually reset (one test per
      field group listed in Req 2, incl. both new keys).
- [ ] Path-field tests: relative, nonexistent, other-owner (simulated via
      injectable ownership checker), world-writable codex_path, dangling
      `default_project` — each resets per spec.
- [ ] 100 concurrent `update` calls serialize without loss (stress test).
- [ ] Secret tripwire test present and green.

## Validation

`swift test --filter 'ConfigStoreTests|SupportPathsTests'` — includes tests
running against an injected temp root.

## Dependencies

01.

## Non-goals

Settings UI, keychain, per-project config files, git-repo validation (25),
hotkey semantic key validation (30/32).

## Design References

DESIGN.md §7.2, Appendix C.2, §9.4, §9.5, §9.2 (T6); ADR-010.
