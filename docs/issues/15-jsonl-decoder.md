# Title

Codex JSONL event decoder with recorded fixtures

## Summary

Implement `HandsfreeAgent/Codex/CodexEventDecoder`: parse the `codex exec
--json` JSONL stream into `AgentEvent`s with strict tolerance rules, plus a
fixtures corpus recorded from real codex runs.

## Context

The event stream is the adapter's source of truth for progress and outcome
(research doc 2026-07-08: envelope `{"type": …}` with `thread.started`,
`turn.started`, `item.*`, `turn.completed`, `turn.failed`; item types
`agent_message`, `reasoning`, `command_execution`, `file_change`,
`mcp_tool_call`, `web_search`, `todo_list`, `error`). Codex evolves quickly:
unknown types MUST degrade gracefully, never crash or abort a turn.

## Scope

- Decoder + wire-model structs + fixtures. Not process handling (14) or
  turn orchestration (18).

## Detailed Requirements

1. `CodexEventDecoder.decode(line: String) -> DecodedLine` where
   `DecodedLine = .event(AgentEvent) | .ignored(reason) | .malformed(error)`.
   Mapping rules:
   - `thread.started` → `.threadStarted(id)` (missing `thread_id` → malformed).
   - `turn.started` → `.turnStarted`.
   - `item.started`/`item.updated` → `.ignored(.progressPhase)` in v1 except
     `command_execution` updates which map to `.item(.commandExecution(…, .running))`
     — narration only needs starts; keep the mapping table in one place.
   - `item.completed` → `.item(mapped AgentItem)` per the payload table:
     `agent_message.text`, `command_execution.command`+`exit_code`→status,
     `file_change` summary/count when present, `web_search.query?`,
     `todo_list`→`.todo(summary: nil)`, `error.message`→`.errorItem`,
     anything else → `.unknown(type)`.
   - `turn.completed` → `.turnCompleted` carrying usage
     (`input_tokens, cached_input_tokens, output_tokens,
     reasoning_output_tokens` — all optional-tolerant).
   - `turn.failed` → `.turnFailed(reason)` (extract best-effort message).
   - stream-level `{"type":"error"}` → `.turnFailed(reason)` equivalent event.
   - Unknown top-level `type` → `.ignored(.unknownEventType(name))`, logged once
     per type per turn (dedup set).
2. Tolerance: decoding uses a two-stage approach — parse envelope
   `{type: String}` first, then payload-specific decode; payload failures on
   known types → `.malformed` (never throw). Item `id` strings are kept for
   dedup (an `item.completed` repeating a seen id → `.ignored(.duplicate)`).
3. Caps: per-turn accepted item count 10 000 (DESIGN §9.5); beyond →
   `.ignored(.itemFlood)` + one warning log.
4. Fixtures (`Tests/Fixtures/jsonl/`):
   - `happy-ping.jsonl` — commit the exact stream captured in the research doc
     probe (2026-07-08).
   - `pre-turn-error-item.jsonl` — the deprecation-warning-as-error-item quirk
     (must NOT fail the turn; research doc).
   - `rich-turn.jsonl` — hand-authored per the documented schema: commands,
     file_change, web_search, final agent_message.
   - `unknown-types.jsonl`, `malformed-lines.jsonl`, `dup-items.jsonl`.
   - `RECORDING.md` — one-page runbook for re-recording fixtures against a
     live codex (command lines, redaction rules) so upgrades refresh evidence.
5. Statistics: decoder exposes `counters` (events, ignored, malformed) for the
   adapter's health check (18: >10 consecutive malformed lines ⇒ turn failure).

## Acceptance Criteria

- [ ] Every fixture decodes with the exact expected `AgentEvent` sequence
      (golden tests, sequences written out in the test).
- [ ] Pre-turn `error` item does not produce `.turnFailed` (quirk covered).
- [ ] Unknown event/item types → `.ignored`/`.unknown` with single dedup log.
- [ ] Duplicate item ids ignored; item-flood cap enforced.
- [ ] Decoder is pure (no I/O) and Sendable.

## Validation

`swift test --filter CodexEventDecoderTests` — 100% of fixture lines asserted.

## Dependencies

12.

## Non-goals

Final-response contract parsing (16), process I/O (14), non-codex agents.

## Design References

DESIGN.md §6.1, §6.2, §9.5; ADR-002; research doc (JSONL schema + quirks).
