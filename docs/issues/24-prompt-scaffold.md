# Title

PromptScaffoldBuilder: versioned prompt templates for every turn kind

## Summary

Implement `HandsfreeCore/Dialogue/PromptScaffoldBuilder` and the scaffold
template resources (`Resources/scaffolds/`): wrap user utterances into the
contract-carrying prompts of DESIGN §6.4 for new tasks, answers, follow-ups,
and approved escalations.

## Context

The scaffold is how the response contract (ADR-005) and the escalation
narrowing (threat T7) are communicated to the agent. Templates are versioned
resources so prompt iteration never requires code changes.

## Scope

- Builder + 4 templates + escaping rules + tests. Not: dispatch (18/28).

## Detailed Requirements

1. Template files (`Resources/scaffolds/`, plain text with `{placeholder}`):
   `new_task.txt`, `answer.txt`, `follow_up.txt`, `approved_action.txt` —
   contents per DESIGN §6.4 (the `<handsfree_contract>` block verbatim for
   new_task; shortened contract reminder for resume variants; the
   `<approved_action>` block naming ONLY the approved action and the
   instruction to perform nothing further at elevated access).
2. Builder API:
   ```swift
   struct PromptScaffoldBuilder {
       func build(kind: TurnKind, utterance: String, locale: SpeechLocale,
                  approvedAction: String?) throws -> String
   }
   enum TurnKind { case newTask, answer, followUp, approvedEscalation }
   ```
   - `approvedEscalation` requires `approvedAction` (else programmer-error
     throw); other kinds forbid it.
3. Escaping: user utterance and approvedAction are inserted inside
   `<user_request>`-style tags; any literal occurrence of the closing tag
   sequences (`</user_request>`, `</user_answer>`, `</user_follow_up>`,
   `</approved_action>`) inside the payload is neutralized by inserting a
   zero-width-free textual break (`< /…>` → documented transformation:
   replace `</` with `<⁄/`? NO — keep it simple and auditable: replace
   the exact closing-tag strings with the same text wrapped in square
   brackets, e.g. `[/user_request]`). The transformation table is a constant
   with tests; utterances are otherwise inserted verbatim (they are trusted
   user speech, not agent output).
4. Locale attribute: `locale="ja-JP" | "en-US"` stamped on the request tag
   (contract tells the agent to answer voice_summary in that language).
5. Template loading: from `Bundle.module` once, cached; missing
   template/unknown placeholder → thrown configuration error at startup
   (fail fast, caught by app boot diagnostics).
6. Version marker: first line of every template `# scaffold-version: 1`
   stripped before use but asserted by tests (future prompt migrations bump it
   and log the active version per turn to transcripts).

## Acceptance Criteria

- [ ] Golden output tests for all 4 kinds × 2 locales (8 files committed as
      expected fixtures).
- [ ] Escaping table test: payload containing every closing tag is neutralized
      and round-trip visible; benign `<` `>` usage passes through.
- [ ] approvedEscalation arg validation both directions.
- [ ] Missing-template and unknown-placeholder failures are typed errors.
- [ ] Templates contain the schema-conformance instruction and voice_summary
      style rules verbatim from DESIGN §6.4 (reviewer diff).

## Validation

`swift test --filter PromptScaffoldTests`.

## Dependencies

01.

## Non-goals

Per-project custom instructions (v2), model-specific prompt tuning, sending
prompts (18).

## Design References

DESIGN.md §6.4, §6.5, §9.2 (T7); ADR-005.
