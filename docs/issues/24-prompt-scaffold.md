# Title

PromptScaffoldBuilder: versioned prompt templates for every turn kind

## Summary

Implement `HandsfreeCore/Dialogue/PromptScaffoldBuilder` and the four
scaffold template resources with their exact text, escaping rules for
untrusted payloads, and an eagerly-validating initializer.

## Context

The scaffold carries the response contract (ADR-005) and the escalation
narrowing (T7) to the agent. STT text is untrusted input (DESIGN §9.5):
payloads must not be able to alter template structure.

## Scope

- Builder + 4 templates + escaping + tests. Not: dispatch (18/28).

## Detailed Requirements

1. Template files (`Sources/HandsfreeCore/Resources/scaffolds/`, first line
   `# scaffold-version: 1`, stripped at load). Exact contents (placeholders
   in braces):
   - `new_task.txt`:
     ```
     <handsfree_contract>
     You are driven by voice. The user cannot easily read long text.
     - Perform the requested work autonomously within your sandbox.
     - If sandbox limits block a required action (network, files outside the
       workspace), stop at a safe point and report status="blocked" with
       blocked_reason and a precise blocked_action.
     - If you need a decision from the user, report status="needs_input"
       with one concise question.
     - Your final message MUST conform to the provided output schema.
     - voice_summary: 1-3 short sentences, same language as the user's
       request, no code, no URLs, no file paths unless essential.
     </handsfree_contract>
     <user_request locale="{locale}">
     {payload}
     </user_request>
     ```
   - `answer.txt`:
     ```
     Continue the task. The user answered your question by voice.
     Remember the handsfree contract: final message conforms to the output
     schema; voice_summary in the user's language, 1-3 short sentences.
     <user_answer locale="{locale}">
     {payload}
     </user_answer>
     ```
   - `follow_up.txt`: same shape as `answer.txt` with
     `Continue the task. The user gave a voice follow-up instruction.` and
     `<user_follow_up locale="{locale}">…`.
   - `approved_action.txt`:
     ```
     The user APPROVED exactly one elevated action for this single turn.
     <approved_action>
     {payload}
     </approved_action>
     Perform that action and nothing further at elevated access, then report
     per the output schema (voice_summary in the user's language).
     ```
     (No locale attribute on `approved_action` — the payload is the
     policy-layer action text, not user speech.)
2. Builder API:
   ```swift
   public struct PromptScaffoldBuilder {
       public init(bundle: Bundle = .module) throws   // eager: loads all 4, checks
                                                      // version markers + placeholder set
       public func build(kind: TurnKind, payload: String,
                         locale: SpeechLocale) throws -> String
   }
   public enum TurnKind { case newTask, answer, followUp, approvedEscalation }
   ```
   `build` substitutes `{locale}` (BCP-47 from `SpeechLocale.rawValue`) and
   `{payload}` exactly once each — single-pass substitution so payload
   content containing `{payload}`/`{locale}` is NOT re-expanded (test).
3. Escaping (untrusted payloads — user speech AND action text): before
   insertion, replace each literal closing-tag sequence
   `</user_request>`, `</user_answer>`, `</user_follow_up>`,
   `</approved_action>` (case-insensitive, whitespace-tolerant inside the
   tag: `</ *tag *>`) with `[/tag]`. Transformation table is a constant with
   tests; benign `<`/`>` usage passes through.
4. Failure modes: missing template, bad version marker, unknown placeholder
   in a template → typed `ScaffoldError` from `init` (fail fast at app
   boot; boot surfaces it per issue 29).
5. Golden fixtures: `Tests/Fixtures/scaffolds/{kind}_{ja|en}.expected.txt`
   (8 files) — exact expected outputs for a fixed payload.

## Acceptance Criteria

- [ ] Golden tests for all 4 kinds × 2 locales match the fixtures.
- [ ] Escaping: payload containing every closing tag (incl. spaced/cased
      variants) neutralized; `{payload}` literal in payload not re-expanded;
      benign angle brackets preserved.
- [ ] Eager init throws on: missing file, wrong version, template with an
      unknown `{placeholder}` (three tests with doctored bundles).
- [ ] `approvedEscalation` output contains the single-action constraint
      sentence verbatim (T7 anchor, asserted).

## Validation

`swift test --filter PromptScaffoldTests`.

## Dependencies

01 (`SpeechLocale`).

## Non-goals

Per-project custom instructions (v2), model-specific tuning, sending prompts
(18), deciding turn kinds (28).

## Design References

DESIGN.md §6.4, §6.5, §9.5 (STT text untrusted), §9.2 (T7); ADR-005.
