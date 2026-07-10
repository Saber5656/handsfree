# Title

Structured logging facade with secret redaction and diagnostic snapshot

## Summary

Implement `HandsfreeCore/Logging`: a thin facade over `os.Logger` with fixed
categories, a redaction preprocessor applied to all payloads before emission,
and a diagnostic snapshot builder with injectable inputs.

## Context

DESIGN §13 fixes subsystem/categories; §9.4 requires defensive masking of
token-like strings before anything is logged or exported. The snapshot
builder must not depend on future issues' concrete types (config is 03, task
index is 27), so it consumes plain-string inputs.

## Scope

- Logging facade, redactor, snapshot builder (DTO-based), tests. No UI button
  (issue 32 wires it), no OSLog persistence changes.

## Detailed Requirements

1. `Log` enum with static wrapped loggers, subsystem
   `AppIdentity.logSubsystem`, categories exactly:
   `audio, stt, tts, dialogue, approval, agent, store, ui` (DESIGN §13).
2. Facade API (exact):
   ```swift
   func debug/info/warning/error(
       _ message: StaticString,
       meta: [String: any CustomStringConvertible] = [:],
       file: StaticString = #fileID, line: UInt = #line)
   ```
   Behavior: metadata is serialized as sorted `key=value` pairs joined by
   spaces; **both keys and values pass `Redactor.redact` first**; the
   resulting single string is logged with `privacy: .public` appended to the
   constant message. Free-form interpolation into `message` is impossible by
   API shape (StaticString). An injectable sink protocol (`LogSink`, default
   = os.Logger) makes emission testable.
3. `Redactor.redact(_ s: String) -> String` — masks each match to
   `<first4>…[redacted]`. Exact pattern list (one table + corpus test):
   - `sk-[A-Za-z0-9_\-]{8,}`
   - `ghp_[A-Za-z0-9]{20,}` · `gho_[A-Za-z0-9]{20,}` · `ghu_[A-Za-z0-9]{20,}`
     · `ghs_[A-Za-z0-9]{20,}` · `github_pat_[A-Za-z0-9_]{20,}`
   - `AKIA[0-9A-Z]{16}`
   - `xox[baprs]-[A-Za-z0-9\-]{10,}`
   - `eyJ[A-Za-z0-9_\-]{20,}\.[A-Za-z0-9_\-]{10,}` (JWT-like)
   - `-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----`
   - `(?i)bearer\s+[A-Za-z0-9._\-]{16,}`
4. Input cap: inputs over 64 KB are truncated FIRST (so masking always sees
   the full retained prefix) and the exact marker `"…[truncated:64KB]"` is
   appended after redaction. O(n) per pattern.
5. `DiagnosticSnapshot.build(input: DiagnosticSnapshotInput,
   logReader: any LogTailReading) -> String` where
   `DiagnosticSnapshotInput { appVersion, macosVersion, codexVersion: String?,
   configJSON: String, taskSummary: String }` (all plain strings supplied by
   the caller — issue 32 assembles them) and `LogTailReading.tail(max: Int)
   throws -> [String]` (production implementation on `OSLogStore` filtered to
   our subsystem, last ≤ 200 lines; failures caught). Every section degrades
   to `"<unavailable: reason>"`; the whole output passes `Redactor.redact`
   once more at the end.

## Acceptance Criteria

- [ ] All eight categories exist as `Log.<category>`.
- [ ] Redaction corpus: ≥ 12 positive (one per pattern incl. all gh
      variants), ≥ 6 negative (e.g. `skiing-2026`, `Bearer` as prose word
      followed by short token) — all pass.
- [ ] Multi-line private key fully masked.
- [ ] 64 KB cap: over-limit input truncated with the exact marker, tokens
      inside the retained prefix still masked.
- [ ] Sink test: token-bearing metadata never reaches the sink unredacted;
      keys are redacted too.
- [ ] Snapshot: healthy path renders all sections; a throwing `LogTailReading`
      yields the degraded marker; planted token in `configJSON` is masked.
- [ ] `grep -rn "print(" Sources/ --include=*.swift` → no output (in PR).

## Validation

`swift test --filter 'RedactorTests|LogFacadeTests|DiagnosticSnapshotTests'`.

## Dependencies

01.

## Non-goals

Settings UI wiring (32), file log persistence, remote telemetry (forbidden),
transcript redaction call sites (27 consumes `Redactor`).

## Design References

DESIGN.md §13, §9.4; ADR-010.
