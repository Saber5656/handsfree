# Title

Phrase dictionaries (ja/en) and IntentMatcher

## Summary

Author the complete v1 phrase dictionaries (`Resources/phrases/ja.json`,
`en.json`) including all narration/approval/announcement templates, and
implement the `IntentMatcher` pipeline that turns finalized STT text into
intents under a context-provided active-intent set.

## Context

R9 mandates dictionary-driven bilingual phrases: adding a locale or synonym
must be data-only. Matching rules are security-relevant (DESIGN §5.2): nonce
digits and the approve keyword match exactly, everything else may fuzz within
bounds. The intent taxonomy is fixed in DESIGN §5.2.

## Scope

- Dictionary JSONs (complete — including the template set DESIGN Appendix B
  marks as "full set enumerated in the phrase-table issue"), schema, matcher,
  tests. Not: FSM consumption (22), approval flow (21).

## Detailed Requirements

1. Dictionary schema exactly per DESIGN Appendix B (`version`, `locale`,
   `intents`, `prefixes`, `digits`, `approve_keyword`, `templates`) plus
   `fillers` (list of leading/trailing filler tokens: ja 「えっと」「あの」
   「うーん」; en "um", "uh", "okay so") and `sentence_prefix_politeness`
   (tokens strippable anywhere: ja 「お願い」「ください」 — used only for
   command intents, never free text).
2. Template keys (complete set; both locales must define every key — parity
   test): `narration.command`, `narration.command_done`, `narration.files`,
   `narration.web_search`, `narration.started`, `announce.session_start`,
   `announce.session_end`, `announce.dispatch_confirm`, `announce.dispatched`,
   `announce.result_ok`, `announce.result_failed`, `announce.result_cancelled`,
   `announce.needs_input_prefix`, `announce.fallback_summary_notice`,
   `announce.continue_prompt`, `announce.task_pending_single`,
   `announce.task_pending_multi`, `announce.task_completed_bg`,
   `approval.announce`, `approval.approved`, `approval.denied`,
   `approval.timeout`, `approval.retry`, `error.not_understood`,
   `error.stt_unavailable`, `error.agent_preflight`, `error.spawn_failed`,
   `error.turn_timeout`, `help.text`. Placeholders use `{name}` syntax;
   a render function substitutes and refuses unknown placeholders (test).
3. `IntentMatcher.match(text: String, context: MatchContext) -> MatchResult`
   where `MatchContext` carries: active intent set (from FSM state), locale,
   whether a task-selection list is offered, known project names (from 25).
   `MatchResult = .intent(Intent) | .freeText(String) | .noMatch`.
4. Pipeline (DESIGN §5.2, in order): NFKC normalize → lowercase (locale-aware)
   → strip punctuation `。、．，!?！？.,;:「」()（）` → strip fillers →
   (a) digits/approve: exact-only match when `approve_echo` is active —
   pattern `approve_keyword d1 d2` with digits from the digit table, spaces
   optional; (b) exact intent lookup; (c) prefix intents
   (`switch_project`, `task_select`) — prefix + remainder resolved (project
   names matched here with the same normalizer; numbers parse from digit
   table words AND arabic numerals); (d) bounded fuzzy for command intents:
   Levenshtein ≤ ⌊len/4⌋, ties → longest alias wins; (e) free text (only if
   the context allows `new_task`/`follow_up`/`answer`), else `.noMatch`.
5. Guarantees: matching is pure, < 10 ms for 200-char input (perf test);
   `approve_echo` can NEVER result from fuzzy or partial input (adversarial
   tests: 「承認 4」, 「承認 よん きゅう じゃなくて」, "confirm 4 9 maybe" →
   `.noMatch`/free-text depending on context; trailing extra tokens invalidate
   an echo).
6. Build-time schema validation: a unit test decodes both dictionaries against
   the schema struct and asserts template-key parity + digit-table
   completeness (0–9) + non-empty alias lists.

## Acceptance Criteria

- [ ] Golden corpus ≥ 60 cases per locale covering every intent, fillers,
      politeness, fuzzy hits, fuzzy-too-far misses, digits (word + numeral),
      code-switched project names (ja utterance with ASCII project name).
- [ ] Adversarial echo suite (≥ 10 cases/locale) proves exact-only nonce rules.
- [ ] Template parity + render tests (unknown placeholder → error).
- [ ] Matcher perf test < 10 ms (DESIGN §15).
- [ ] Dictionaries load from `Bundle.module` in the assembled app (05 smoke).

## Validation

`swift test --filter IntentMatcherTests PhraseDictionaryTests`.

## Dependencies

01.

## Non-goals

Wake word phrases, per-user custom phrases (v2), UI strings localization
(issue 29/32 `.strings` files — distinct from these spoken-phrase tables).

## Design References

DESIGN.md §5.2, §5.3, Appendix B, §9.2 (T1), §15; R9.
