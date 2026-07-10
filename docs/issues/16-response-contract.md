# Title

Structured response contract: schema resource, parser, fallback chain

## Summary

Implement `HandsfreeAgent/Contract`: the bundled `response-schema.json`
(DESIGN Appendix C.1), schema-equivalent native validation, the parsing
chain of DESIGN §6.3, and the fallback summarizer — producing a
`ParsedResponse` that issue 18 embeds into `TurnOutcome`.

## Context

The contract is both the TTS shortening layer (ADR-005) and the control-flow
signal for needs-input/escalation (§6.3). It is a trust boundary: agent
output is untrusted (T3). DESIGN §9.5 requires validation equivalent to the
schema — implemented natively (no third-party JSON-Schema engine, ADR-010).
Module direction: `HandsfreeAgent` must NOT import `HandsfreeCore`, so this
issue implements its own code-point-safe truncation; full speech
sanitization stays at the Core speech boundary (20/28).

## Scope

- Schema resource, `ResponseContractParser`, native validation, fallback
  reducer, truncation helper. `AgentResponse` struct already exists (12).

## Detailed Requirements

1. `Sources/HandsfreeAgent/Resources/response-schema.json`: canonical schema
   per DESIGN Appendix C.1. Drift guard: a unit test decodes the resource
   and asserts — for every property — type, enum values, maxLength, the
   required list, and `additionalProperties=false`, against constants that
   the validator itself uses (single source in code; the test proves
   resource ↔ validator equivalence, superseding "byte-for-byte" wording).
2. Native validation (`ContractValidator.validate(json: Data) ->
   Result<AgentResponse, ContractViolation>`): top-level object; no unknown
   keys; `status` enum; `voice_summary` non-empty string; nullable string
   types for the rest; `blocked_reason` enum. Length caps applied by
   truncation (not rejection): `voice_summary` 400, `question` 300,
   `blocked_action` 300, `detail` 20 000, `proposed_next_action` 200 —
   code-point-safe (`TruncationHelper`, local to this module, tested with
   CJK + emoji).
3. Parsing chain — precise result table (`ParsedResponse`):
   ```swift
   public struct ParsedResponse: Sendable {
       public let response: AgentResponse
       public let source: Source            // .finalMessage | .lastMessageFile | .fallback
       public let isFallbackSummary: Bool
       public let repairs: [Repair]         // .questionMissing | .blockedFieldsInvalid
   }
   ```
   Steps: (1) try `finalMessageText` if non-empty; (2) on ANY
   `ContractViolation` from step 1, try the `-o` file contents if readable
   and non-empty; (3) on violation again, fallback reducer over the first
   available non-empty raw text (finalMessageText, else file contents, else
   the fixed localizable key `contract.empty_response` handled upstream).
   **Semantic repairs do NOT fall through** — they apply to an otherwise
   valid parse and are recorded in `repairs`:
   - `status=needs_input` with empty/absent `question` → treated as `ok`
     (repair `.questionMissing`, logged by the adapter);
   - `status=blocked` with invalid `blocked_reason` OR empty
     `blocked_action` → treated as `failed(reason: "malformed_blocked")`
     (repair `.blockedFieldsInvalid`) — an unaccountable escalation request
     must never reach the approval engine.
4. Fallback reducer: strip fenced code blocks, inline code, markdown
   links/images (keep link text), URLs, header/list markers; collapse
   whitespace; first 2 sentences (ja 。！？ + latin .!?) up to 280 chars;
   `isFallbackSummary=true`. Deterministic and total (never throws).
5. Property/fuzz test: ≥ 1000 random and mutated inputs across all three
   steps never crash and always yield a `ParsedResponse`.

## Acceptance Criteria

- [ ] Golden tests: valid ok / needs_input / blocked / failed → exact
      structs, `source=.finalMessage`.
- [ ] Step-2 path: invalid final text + valid `-o` file →
      `source=.lastMessageFile`.
- [ ] Repair rules: needs_input-no-question → ok+repair; malformed blocked →
      failed+repair (both with `source` ≠ `.fallback`).
- [ ] Caps: oversize fields truncated at exact code-point-safe lengths
      (multi-byte tests).
- [ ] Fallback reducer: 5 KB markdown-heavy input → ≤ 280 chars, no
      code/URLs, sentence-boundary cut (ja and en).
- [ ] Precedence when both JSON parses fail but both texts exist →
      fallback uses `finalMessageText` (test); file-only case covered.
- [ ] Schema↔validator drift test green; fuzz test green.

## Validation

`swift test --filter 'ResponseContractTests|FallbackSummarizerTests|TruncationHelperTests'`.

## Dependencies

12.

## Non-goals

Speech-side sanitization (20/28 own the TTS boundary), spawn flags (18),
schema version negotiation (known unknown #4 — revisit on breakage).

## Design References

DESIGN.md §6.3 (amended caps note), Appendix C.1, §9.5, §16 (#4); ADR-005,
ADR-010; DESIGN §3.1 (module direction).
