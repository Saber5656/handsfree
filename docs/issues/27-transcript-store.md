# Title

TranscriptStore: session JSONL records, task index, retention sweep

## Summary

Implement `HandsfreeCore/Transcripts/TranscriptStore`: append-only per-session
JSONL files and the `tasks.json` index, with strict permissions, redaction,
retention enforcement, and the privacy switches of DESIGN §7.3.

## Context

Transcripts are the audit trail for approvals (threat T7 evidence) and the
recovery source for tasks (26) — but also the biggest privacy liability
(threat T5). Retention and opt-out must be real, not cosmetic.

## Scope

- Store, record schema, sweep, delete-all. Not: what gets recorded when
  (callers), Settings UI (33).

## Detailed Requirements

1. Layout (DESIGN §7.3):
   `<AppSupport>/Handsfree/transcripts/YYYY/MM/<session-id>.jsonl` and
   `<AppSupport>/Handsfree/tasks.json`. Dirs `0700`, files `0600`
   (same `SupportPaths` as 03). Session id: `<yyyyMMdd-HHmmss>-<4 random hex>`.
2. Record envelope: `{"ts": ISO8601-millis, "type": String, "v": 1, …payload}`.
   Types + payloads (Codable structs, exhaustive):
   `session_meta` (locale, app version, project id/name),
   `utterance` (final text ONLY — never volatile partials),
   `intent` (matched intent name, elided free text length),
   `dispatch` (task id, tier, prompt kind, scaffold version — NOT the full
   prompt; the utterance record already has the user text),
   `agent_item` (task id, item type, payload truncated 512 chars,
   redacted via 04),
   `approval` (task id, tier, nonce digits, attempt texts redacted, decision,
   duration ms),
   `result` (task id, status, voice_summary, isFallback),
   `error` (domain, key, message redacted).
3. Writer: actor with an append queue; each line flushed; write failures
   (disk full) → drop transcript writes for the session, log once, emit a
   one-time in-app warning event (DESIGN §12) — session must keep working.
4. Privacy switches (config, DESIGN §7.3):
   - `store_transcripts=false` OR `retention_days=0`: records go to a
     temp-dir file unlinked at session close (exists only for crash debugging
     during the session; test asserts absence after close).
   - retention_days N: sweep at app launch + every 24 h deletes session files
     with mtime older than N days AND removes empty YYYY/MM dirs. tasks.json
     entries for terminal+acknowledged tasks older than N days are pruned.
5. `deleteAllNow()` — synchronous best-effort removal of all transcript files
   + index rebuild from live tasks only; returns count for UI confirmation.
6. Task index: atomic read/modify/write of `tasks.json` (temp+rename),
   schema per 26's `TaskRecord` (Codable), corruption → `.corrupt-<ts>`
   sidecar + empty index (recovery-safe, mirrors 03's policy).
7. Reader API for the HUD/menu (last session summaries): `recentSessions(limit:)`,
   `records(sessionID:)` — lazy line decoding, tolerant of unknown record
   types (forward compat).

## Acceptance Criteria

- [ ] Permissions asserted on every created dir/file.
- [ ] All record types round-trip; unknown-type lines skipped on read.
- [ ] Volatile-partials guard: API surface has no method accepting non-final
      STT results (type-level: takes `FinalUtterance` newtype).
- [ ] Retention sweep: fixture tree with old/new files → correct deletions,
      empty-dir cleanup, index pruning (virtual clock).
- [ ] `store_transcripts=false` leaves zero files post-session.
- [ ] Disk-full simulation (injected throwing FileHandle) degrades per Req 3.
- [ ] `deleteAllNow` removes everything and preserves live-task index entries.
- [ ] Redactor (04) applied to `agent_item`/`approval`/`error` payloads
      (test with a token-bearing payload).

## Validation

`swift test --filter TranscriptStoreTests`.

## Dependencies

03, 04.

## Non-goals

Transcript viewer UI (menu/HUD show digests only in v1), export formats,
cloud sync (never).

## Design References

DESIGN.md §7.3, §9.2 (T5/T7), §9.4, §12.
