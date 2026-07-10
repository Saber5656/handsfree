# Title

Structured logging facade with secret redaction and diagnostic snapshot

## Summary

Implement `HandsfreeCore/Logging`: a thin facade over `os.Logger` with fixed
categories, a redaction preprocessor applied to all interpolated payloads, and
a diagnostic snapshot builder used later by Settings ("Copy diagnostic
snapshot").

## Context

DESIGN §13 fixes subsystem/categories; §9.4 requires defensive masking of
token-like strings before anything is logged or exported. Transcripts (27) and
bug reports must never leak credentials that appear in agent output or user
utterances.

## Scope

- Logging facade, redactor, snapshot builder, tests. No UI button (issue 32).

## Detailed Requirements

1. `Log` enum with static `os.Logger` instances, subsystem
   `AppIdentity.logSubsystem`, categories exactly:
   `audio, stt, tts, dialogue, approval, agent, store, ui` (DESIGN §13).
2. API shape: `Log.agent.info("spawned", meta: ["pid": pid])` — a small wrapper
   `func info/debug/warning/error(_ message: StaticString-like…, meta: [String: CustomStringConvertible])`
   where **meta values pass through the redactor** and are logged `%{public}s`
   after redaction; free-form message strings are discouraged by the API shape
   (message is a constant, data goes in meta).
3. `Redactor.redact(_ s: String) -> String` masking (keep first 4 chars + `…`):
   - `sk-[A-Za-z0-9_\-]{8,}` (OpenAI-style), `ghp_[A-Za-z0-9]{20,}`,
     `github_pat_[A-Za-z0-9_]{20,}`, `gho_/ghu_/ghs_` variants,
     `AKIA[0-9A-Z]{16}`, `xox[baprs]-[A-Za-z0-9\-]{10,}`,
     `eyJ[A-Za-z0-9_\-]{20,}\.[A-Za-z0-9_\-]{10,}` (JWT-ish),
     `-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----`,
     `(?i)bearer\s+[A-Za-z0-9._\-]{16,}`.
   - Patterns live in one table with unit-test corpus (positive + negative
     cases — e.g., `skiing-2026` must NOT be masked).
4. Redactor is exported for reuse by transcript store (27) and diagnostic
   snapshot; complexity O(n) per pattern, input capped at 64 KB per call
   (longer inputs truncated first, noted with a marker).
5. `DiagnosticSnapshot.build(config:, taskIndex:, logTail:)` returning a single
   redacted string: app version (from `VERSION` resource), macOS version,
   codex version (param), config JSON (already secret-free by design, still
   redacted), last ≤200 log lines fetched via `OSLogStore` for our subsystem,
   task index summary. Must never throw on partial failure — sections degrade
   to `"<unavailable: reason>"`.

## Acceptance Criteria

- [ ] All eight categories exist and compile as `Log.<category>`.
- [ ] Redaction corpus test: ≥12 positive, ≥6 negative cases pass.
- [ ] Multi-line private key block fully masked in one pass.
- [ ] Snapshot builder produces all sections on a healthy system and degrades
      gracefully when `OSLogStore` access fails (simulated).
- [ ] No `print()` anywhere in production targets (lint grep in Validation).

## Validation

- `swift test --filter RedactorTests` and `--filter DiagnosticSnapshotTests`.
- `grep -rn "print(" Sources/ --include=*.swift` returns nothing
  (documented in PR; earcons/dev tools excluded — none exist yet).

## Dependencies

01.

## Non-goals

Settings UI wiring, file-based log persistence, remote telemetry (forbidden).

## Design References

DESIGN.md §13, §9.4; ADR-010.
