# Title

Codex JSONL event decoder with recorded fixtures

## Summary

Implement `HandsfreeAgent/Codex/CodexEventDecoder`: a pure, side-effect-free
decoder from `codex exec --json` JSONL lines to **wire-level events**
(`CodexWireEvent`), with strict tolerance rules and a fixtures corpus. The
adapter (18) maps wire events to the public `AgentEvent` model and assembles
`TurnOutcome`.

## Context

The event stream is the adapter's source of truth (research doc: envelope
`{"type": …}`; event types `thread.started`, `turn.started`, `item.*`,
`turn.completed`, `turn.failed`; item types `agent_message`, `reasoning`,
`command_execution`, `file_change`, `mcp_tool_call`, `web_search`,
`todo_list`, `error`). Codex evolves quickly: unknown types MUST degrade
gracefully. Contract parsing and outcome assembly are explicitly NOT here
(16/18) — hence the wire-event layer.

## Scope

- Decoder + wire model + fixtures + golden expected sequences. No process
  I/O (14), no `AgentEvent` mapping (18).

## Detailed Requirements

1. Wire model and API (pure, `Sendable`, no imports beyond Foundation):
   ```swift
   public enum CodexWireEvent: Equatable, Sendable {
       case threadStarted(id: String)
       case turnStarted
       case itemCompleted(WireItem)
       case turnCompleted(usage: WireUsage?)
       case turnFailed(reason: String)
       case streamError(message: String)
   }
   public struct WireItem: Equatable, Sendable {
       public let id: String?
       public let kind: WireItemKind
   }
   public enum WireItemKind: Equatable, Sendable {
       case agentMessage(text: String)
       case commandExecution(command: String, exitCode: Int?)
       case fileChange(summary: String?, fileCount: Int?)
       case webSearch(query: String?)
       case todoList
       case errorItem(message: String)       // NON-terminal (codex quirk)
       case reasoning
       case unknown(type: String)
   }
   public struct DecodeState { … }            // per-turn dedup/counter state, caller-owned
   public enum DecodedLine: Equatable, Sendable {
       case event(CodexWireEvent)
       case ignored(IgnoreReason)   // .progressPhase | .duplicateCompleted(id)
                                    // | .unknownEventType(name, firstOccurrence: Bool)
                                    // | .itemFlood | .lineTooLong
       case malformed(reason: String)
   }
   public struct CodexEventDecoder {
       public func decode(line: String, state: inout DecodeState) -> DecodedLine
   }
   ```
   No logging inside the decoder: `unknownEventType` carries
   `firstOccurrence` (computed from a per-state dedup set) so the ADAPTER
   logs once per type per turn. `state` resets are the caller's job at
   `turnStarted`.
2. Wire-payload mapping table (exact JSON paths; anything missing/mistyped
   on a KNOWN type ⇒ `.malformed`, except where marked optional):
   | Line | Fields | Result |
   |---|---|---|
   | `type=thread.started` | `thread_id: String` (required) | `.threadStarted` |
   | `type=turn.started` | — | `.turnStarted` |
   | `type=item.started/.updated` | — | `.ignored(.progressPhase)` (v1 narrates completions only) |
   | `type=item.completed`, `item.type=agent_message` | `item.text: String` | `.agentMessage` |
   | …`item.type=command_execution` | `item.command: String` (required), `item.exit_code: Int?` | `.commandExecution` |
   | …`item.type=file_change` | `item.summary: String?`, `item.file_count: Int?` — if neither present, summary falls back to `""` | `.fileChange` |
   | …`item.type=web_search` | `item.query: String?` | `.webSearch` |
   | …`item.type=todo_list` | — | `.todoList` |
   | …`item.type=error` | `item.message: String` | `.errorItem` |
   | …`item.type=reasoning` | — | `.reasoning` |
   | …other `item.type` | — | `.unknown(type)` |
   | `type=turn.completed` | `usage.{input_tokens,cached_input_tokens,output_tokens,reasoning_output_tokens}: Int?` (all optional) | `.turnCompleted` |
   | `type=turn.failed` | best-effort reason: first present of `error.message`, `message`, `reason`; else `"turn failed"` | `.turnFailed` |
   | `type=error` | `message: String?` (fallback `"stream error"`) | `.streamError` |
   | other `type` | — | `.ignored(.unknownEventType)` |
   Field names are pinned against the committed fixtures; `RECORDING.md`
   (Req 4) is the procedure for re-pinning on codex upgrades.
3. Caps and dedup:
   - Defensive line cap: lines > 1 MB → `.ignored(.lineTooLong)` (primary
     enforcement is 14's drop; this is belt-and-braces).
   - Dedup applies to `item.completed` ONLY: a repeated completed `id` →
     `.ignored(.duplicateCompleted)`. Started/updated phases never mark ids
     as seen.
   - Accepted `item.completed` count per turn capped at 10 000; beyond →
     `.ignored(.itemFlood)`.
4. Fixtures (`Tests/Fixtures/jsonl/`), each with a sibling `.expected` file
   listing the exact `DecodedLine` sequence (golden oracle checked in):
   - `happy-ping.jsonl` — normalized from the 2026-07-08 live probe (the
     probe's truncated deprecation message is completed to a full literal;
     header comment notes the normalization).
   - `pre-turn-error-item.jsonl` — deprecation-warning-as-error-item quirk
     (must decode as `.errorItem`, not a failure).
   - `rich-turn.jsonl` — hand-authored per Req 2 table: commands (running +
     exit codes), file_change, web_search, todo_list, reasoning, final
     agent_message, turn.completed with usage.
   - `unknown-types.jsonl`, `malformed-lines.jsonl`, `dup-items.jsonl`,
     `oversize-line.jsonl` (1 MB+ single line).
   - `RECORDING.md` — runbook to re-record against a live codex (commands,
     redaction rules, how to regenerate `.expected` files).

## Acceptance Criteria

- [ ] Every fixture line's `DecodedLine` matches its `.expected` entry
      (golden test iterates all fixtures).
- [ ] Pre-turn `error` item decodes as `.errorItem` (non-terminal).
- [ ] Unknown event/item types → `.ignored`/`.unknown` with correct
      `firstOccurrence` flags; state reset at `turnStarted` re-arms them.
- [ ] Duplicate completed ids ignored; started→completed lifecycle NOT
      treated as duplicate; item-flood cap enforced.
- [ ] Oversize line → `.ignored(.lineTooLong)`.
- [ ] Decoder is pure (Foundation-only imports; no logging; deterministic).

## Validation

`swift test --filter CodexEventDecoderTests`.

## Dependencies

12 (module placement; no type coupling — wire model is decoder-local).

## Non-goals

`AgentEvent` mapping and outcome assembly (18), contract parsing (16),
non-codex agents.

## Design References

DESIGN.md §6.1 (amended), §6.2, §9.5; ADR-002; research doc (JSONL schema,
probe quirks).
