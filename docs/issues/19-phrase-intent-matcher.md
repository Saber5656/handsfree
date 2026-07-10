# Title

Phrase dictionaries (ja/en) and IntentMatcher

## Summary

Author the complete v1 phrase dictionaries
(`Sources/HandsfreeCore/Resources/phrases/{ja,en}.json`) including every
template key, and implement the `IntentMatcher` pipeline that turns finalized
STT text into intents under a context-provided active-intent set.

## Context

R9 mandates dictionary-driven bilingual phrases. Matching rules are
security-relevant (DESIGN §5.2/Appendix B): nonce digits and the approve
keyword match exactly; ja digit vocabulary excludes homophones (7=なな,
never しち). Politeness stripping applies only to free-text extraction,
never before exact alias lookup (DESIGN Appendix B, amended).

## Scope

- Dictionary JSONs, schema struct, matcher, tests. Not: FSM consumption
  (22), approval flow (21), project resolution logic (25 — but the
  project-error template keys ARE defined here).

## Detailed Requirements

1. Dictionary schema per DESIGN Appendix B: `version`, `locale`, `intents`,
   `prefixes`, `digits`, `approve_keyword`, `templates`, plus `fillers`
   (ja: えっと/あの/うーん; en: um/uh/"okay so") and `politeness` tokens
   (ja: お願い/ください; en: please) — politeness applies ONLY to free-text
   command extraction (Req 4e), never to exact alias lookup.
2. Template keys (complete closed set; both locales define every key —
   parity test): `narration.command`, `narration.command_done`,
   `narration.files`, `narration.web_search`, `narration.todo_update`,
   `narration.started`, `announce.session_start`, `announce.session_end`,
   `announce.dispatch_confirm`, `announce.dispatched`, `announce.result_ok`,
   `announce.result_failed`, `announce.result_cancelled`,
   `announce.needs_input_prefix`, `announce.fallback_summary_notice`,
   `announce.continue_prompt`, `announce.task_pending_single`,
   `announce.task_pending_multi`, `announce.task_completed_bg`,
   `announce.bg_continue`, `approval.announce`, `approval.approved`,
   `approval.denied`, `approval.timeout`, `approval.retry`,
   `approval.tier3_disabled`, `error.not_understood`,
   `error.stt_unavailable`, `error.agent_preflight`, `error.spawn_failed`,
   `error.turn_timeout`, `error.project_not_found`,
   `error.project_invalid_path`, `error.project_not_git`,
   `error.project_ambiguous`, `error.registry_empty`,
   `error.too_many_tasks`, `error.quitting_drain`, `contract.empty_response`,
   `help.text`. Placeholders `{name}`; the render function rejects unknown
   placeholders (typed error, tested).
3. `IntentMatcher.match(text: String, context: MatchContext) -> MatchResult`
   with `MatchContext { active: Set<IntentKind>, locale: SpeechLocale,
   selectionOffered: Bool, knownProjectNames: [String] }` —
   `knownProjectNames` is a plain string list (no ProjectRegistry
   dependency; 25 supplies it at runtime).
   `MatchResult = .intent(Intent) | .freeText(String) | .noMatch`.
4. Pipeline (in order):
   a. NFKC normalize → locale-aware lowercase → strip punctuation
      `。、．，!?！？.,;:「」()（）` → strip leading/trailing fillers.
   b. If `approve_echo` is active: exact-only pattern
      `approve_keyword d1 d2` (digits from the digit table, arabic numerals
      also accepted, spaces optional). ANY extra tokens ⇒ not an echo. No
      fuzzing, ever.
   c. Exact intent alias lookup (politeness NOT stripped yet).
   d. Prefix intents (`switch_project`, `task_select`): prefix + remainder;
      project names matched against `knownProjectNames` with the same
      normalizer; task numbers parse from digit words AND numerals.
   e. Bounded fuzzy for command intents only: Levenshtein ≤ ⌊len/4⌋; ties →
      longest alias wins. Politeness tokens are stripped before this step
      and before free-text fallthrough.
   f. Free text (only if context allows `new_task`/`follow_up`/`answer`),
      else `.noMatch`.
5. Guarantees: pure; < 10 ms for 200-char input (perf test); ja digit table
   MUST NOT contain しち (schema test); a resource-load unit test opens both
   dictionaries via `Bundle.module` (assembled-app smoke stays with 05/38).

## Acceptance Criteria

- [ ] Golden corpus ≥ 60 cases/locale: every intent, fillers, politeness
      (incl. bare 「お願い」 in confirming context → NOT `yes`), fuzzy
      hits/misses, digits (word + numeral), code-switched project names.
- [ ] Adversarial echo suite ≥ 12 cases/locale: 「承認 4」, digits reversed,
      「承認 しち きゅう」 (must fail), extra-token suffixes, fuzzy-distance
      keyword variants — all rejected.
- [ ] Template parity + render tests (unknown placeholder → error; every key
      of Req 2 present in both locales).
- [ ] Digit-table schema tests (0–9 complete; しち absent).
- [ ] Matcher perf < 10 ms (DESIGN §15).
- [ ] `Bundle.module` load test green.

## Validation

`swift test --filter 'IntentMatcherTests|PhraseDictionaryTests'`.

## Dependencies

01.

## Non-goals

Wake word, per-user custom phrases (v2), UI `.strings` localization (29/32),
who calls the matcher (22/28).

## Design References

DESIGN.md §5.2, §5.3, Appendix B (amended), §9.2 (T1), §15; R9.
