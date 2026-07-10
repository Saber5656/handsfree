# Title

TranscriptStore: session JSONL records, task index primitives, retention

## Summary

Implement `HandsfreeCore/Transcripts/TranscriptStore`: append-only
per-session JSONL with comprehensive redaction, the atomic `tasks.json`
index primitives consumed by 26, retention enforcement, and the privacy
switches of DESIGN §7.3.

## Context

Transcripts are the audit trail for approvals (T7 evidence) and the recovery
source for tasks — and the biggest privacy liability (T5). Every untrusted
string is redacted at persist time; opt-outs must be real.

## Scope

- Store actor, record schema, index primitives, sweep, delete APIs. Not:
  what/when to record (28 calls it), Settings UI (33).

## Detailed Requirements

1. Layout: `<AppSupport>/Handsfree/transcripts/YYYY/MM/<session-id>.jsonl`
   + `<AppSupport>/Handsfree/tasks.json`; dirs `0700`, files `0600`
   (`SupportPaths` from 03, injectable root). Session id
   `<yyyyMMdd-HHmmss>-<4 hex>`.
2. API (exact):
   ```swift
   public actor TranscriptStore {
       public init(paths: SupportPaths, config: ConfigSnapshotProviding,
                   redactor: Redacting, clock: any Clock<Duration>) throws
       public func beginSession(meta: SessionMeta) async -> SessionHandle
       public func append(_ record: TranscriptRecord, to: SessionHandle) async
       public func endSession(_ h: SessionHandle) async
       // task index primitives (26 owns the TaskRecord semantics):
       public func readTaskIndex() async -> TaskIndexFile
       public func writeTaskIndex(_ f: TaskIndexFile) async throws   // temp+rename atomic
       // maintenance:
       public func sweepRetention() async -> SweepResult
       public func deletionSummary() async -> DeletionSummary        // dry-run counts (33 uses)
       public func deleteAllNow() async -> Int                        // returns deleted file count
       public func recentSessions(limit: Int) async -> [SessionSummary]
       public func records(sessionID: String) async -> [TranscriptRecord]  // tolerant reader
       public var events: AsyncStream<TranscriptStoreEvent>
           // .writesDisabled(reason: WriteFailureReason)  — emitted at most once per session
   }
   public struct TaskIndexFile: Codable { public var version: Int; public var entries: [Data] }
   // entries are opaque per-record JSON blobs; 26 encodes/decodes TaskRecord —
   // the store guarantees atomicity + corruption recovery only.
   ```
3. Record envelope `{"ts": ISO8601-millis, "type": String, "v": 1, …}`;
   Codable structs for: `session_meta` (locale, app version, project),
   `utterance` (final text only — the API takes a `FinalUtterance` newtype
   so volatile partials are unrepresentable), `intent` (name + free-text
   length only), `dispatch` (task id, tier, prompt kind, scaffold version —
   never the full prompt), `agent_item` (task id, item type, payload ≤ 512),
   `approval` (task id, tier, nonce digits, attempt texts, decision,
   duration ms), `result` (task id, status, voice_summary, isFallback),
   `error` (domain, key, message).
   **Redaction: EVERY free-text field of EVERY record type passes
   `Redactor.redact` at append time** — utterance text, agent_item payload,
   approval attempts, result voice_summary, error message (tests plant
   tokens in each).
4. Privacy switches: `store_transcripts=false` OR `retention_days=0` ⇒
   session records go to a temp-dir file unlinked at `endSession` (asserted:
   zero files under transcripts/ after close in BOTH modes). Otherwise
   `sweepRetention` (at init + every 24 h via the injected clock): delete
   session files with `mtime < now − N×86400 s` (files exactly at the
   boundary are KEPT), remove empty YYYY/MM dirs, prune index entries whose
   task is terminal+acknowledged and older than N days.
5. Write failures (disk full etc., injected throwing writer in tests): drop
   transcript writes for the rest of the session, emit
   `.writesDisabled(.diskFull)` exactly once, keep the session alive
   (DESIGN §12). Index write failures throw to the caller (26 owns its
   retry policy).
6. Corruption: unparseable `tasks.json` → rename `.corrupt-<ts>`, return
   empty index (mirrors 03). Unknown record types skipped on read (forward
   compat).

## Acceptance Criteria

- [ ] Permissions asserted on every created dir/file.
- [ ] All record types round-trip; unknown types skipped on read.
- [ ] `FinalUtterance` newtype prevents partial persistence (compile-level).
- [ ] Redaction test per record type (planted `sk-…`/`ghp_…` masked).
- [ ] Retention: fixture tree old/new/boundary → correct deletions, empty-dir
      cleanup, index pruning (virtual clock).
- [ ] BOTH opt-out modes leave zero transcript files post-session.
- [ ] Disk-full: single `.writesDisabled` event, session continues.
- [ ] `deletionSummary` counts match a subsequent `deleteAllNow`; live task
      index entries preserved.
- [ ] Index atomicity: torn-write simulation (temp file left behind) does
      not corrupt reads.

## Validation

`swift test --filter TranscriptStoreTests`.

## Dependencies

03, 04.

## Non-goals

TaskRecord semantics (26), transcript viewer UI, export formats, cloud sync
(never).

## Design References

DESIGN.md §7.3, §9.2 (T5/T7), §9.4, §12.
