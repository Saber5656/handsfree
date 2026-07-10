# Title

Structured response contract: schema resource, parser, fallback chain

## Summary

Implement `HandsfreeAgent/Contract`: the bundled `response-schema.json`
(DESIGN Appendix C.1), the `AgentResponse` struct, the three-step parsing
chain of DESIGN §6.3, and field-level validation with caps.

## Context

This contract is simultaneously the TTS shortening layer (ADR-005) and the
control-flow signal for needs-input/escalation (DESIGN §6.3). It is a trust
boundary: agent output is untrusted input (threat T3) and must be validated
and capped before anything downstream sees it.

## Scope

- Schema resource, parsing, validation, fallback summarizer. Not: TTS
  sanitization internals (uses issue 20's `SpeechTextSanitizer` caps), spawn
  flags (18).

## Detailed Requirements

1. `Resources/response-schema.json` (owned by `HandsfreeAgent` target):
   byte-for-byte the schema of DESIGN Appendix C.1. A unit test parses the
   resource and asserts required fields + enum values so DESIGN and resource
   cannot drift silently.
2. `AgentResponse` struct (fills the shell from issue 12):
   `status: ResponseStatus (.ok/.needsInput/.blocked/.failed)`,
   `voiceSummary: String`, `question: String?`, `blockedReason:
   BlockedReason? (.needsNetwork/.needsFullAccess/.needsOutOfWorkspace)`,
   `blockedAction: String?`, `detail: String?`, `proposedNextAction: String?`.
3. Parsing chain `ResponseContractParser.parse(finalMessageText: String?,
   lastMessageFile: URL?) -> ParsedResponse`:
   1. `finalMessageText` (the last `agent_message` item) as JSON.
   2. Else the `-o` file contents as JSON.
   3. Else fallback: `ParsedResponse.fallback(summary:…)` — rule-based
      reduction of raw text: strip fenced code blocks, inline code, markdown
      links/images (keep link text), URLs, headers/list markers; collapse
      whitespace; take the first 2 sentences (ja sentence delimiters 。！？
      and latin .!?) up to 280 chars; `isFallbackSummary=true` (orchestrator
      prefixes a spoken auto-summary notice, 28).
   Each step's failure is logged (`Log.agent`) with a reason.
4. Semantic validation (applied to steps 1–2 results; violation ⇒ fall through
   to next step):
   - required: `status` ∈ enum, non-empty `voice_summary`;
   - `status=needs_input` ⇒ `question` non-empty (else downgrade: treat as
     `ok`, log);
   - `status=blocked` ⇒ `blocked_reason` valid AND `blocked_action` non-empty
     (else treat as `failed` with reason `malformed_blocked` — an
     unaccountable escalation request must never reach the approval engine);
   - caps enforced by truncation at code-point boundaries:
     `voice_summary` 400, `question` 300, `blocked_action` 300, `detail`
     20 000, `proposed_next_action` 200 (DESIGN §9.5).
5. JSON Schema validation note: full JSON-Schema evaluation is NOT
   implemented (no third-party deps); the semantic validation above is the
   normative gate. Document this in code comments (the schema file's job is
   to constrain codex's output at generation time).
6. Determinism: parser is pure; property-based test with random garbage
   never crashes and always lands in `.fallback`.

## Acceptance Criteria

- [ ] Golden tests: valid ok / needs_input / blocked / failed payloads parse
      to exact structs.
- [ ] Downgrade rules (needs_input w/o question; blocked w/o action) behave as
      specified with logged reasons.
- [ ] Caps: oversize fields truncated at correct lengths (multi-byte safe —
      test with Japanese + emoji).
- [ ] Fallback reducer: markdown-heavy 5 KB input → ≤ 280 chars, no code/URLs,
      sentence-boundary cut (ja and en cases).
- [ ] Schema resource ↔ DESIGN drift test present.
- [ ] Fuzz test (≥1000 random/mutated inputs) reaches fallback without throwing.

## Validation

`swift test --filter ResponseContractTests FallbackSummarizerTests`.

## Dependencies

12, 20 (truncation/sanitizer utilities).

## Non-goals

Speech-side sanitization (20 owns stripping for TTS), prompt scaffold (24),
schema versioning/negotiation with codex (known unknown #4 — revisit on break).

## Design References

DESIGN.md §6.3, Appendix C.1, §9.5, §16 (#4); ADR-005.
