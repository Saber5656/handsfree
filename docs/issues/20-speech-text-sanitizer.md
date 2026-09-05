# Title

SpeechTextSanitizer: the validation layer for everything spoken or notified

## Summary

Implement `HandsfreeCore/Dialogue/SpeechTextSanitizer`: the single
sanitation + truncation utility applied to any agent-derived text before it
reaches TTS, notifications, or approval announcements.

## Context

Agent output is untrusted (T3, DESIGN §9.2/§9.5). Per the amended DESIGN
§9.2 T3 row, keyword-based "imperative content" filtering is deliberately
NOT part of the defense — the effective mitigations are the unpredictable
nonce, template-only announcements, and the approval earcon. This component
guarantees the *content* is inert (no control/markup/URL smuggling) and
speakable.

## Scope

- One pure utility + exhaustive tests. Consumers: 21, 23, 28, 35.
  (Issue 16 has its own module-local truncation; see DESIGN §3.1 direction.)

## Detailed Requirements

1. API:
   ```swift
   public enum SpeechTextSanitizer {
       /// Full pipeline; appends the locale ellipsis suffix when truncated.
       public static func sanitize(_ s: String, maxLength: Int, locale: SpeechLocale) -> String
       /// Suffix-free, code-point-safe, sentence/word-aware truncation.
       public static func truncate(_ s: String, max: Int) -> String
   }
   ```
   `sanitize` output length ≤ `maxLength` INCLUDING the suffix
   (ja 「…以下省略」 / en "… truncated").
2. Sanitize pipeline (order matters; each step unit-tested):
   1. Unicode NFC.
   2. Remove control chars (C0/C1; `\n` → space), zero-width chars
      (ZWSP/ZWNJ/ZWJ/BOM), bidi controls (U+202A–202E, U+2066–2069).
   3. Strip tag-like markup with a bounded scanner (not a single regex):
      a `<` opens a candidate tag; if a `>` follows within 4096 chars the
      span is removed (covers `<speak …>` with long attributes); an
      unclosed `<` beyond that bound is kept literally. Then strip markdown
      emphasis/backticks/fences, `#` headers, list markers at line starts.
   4. Replace URLs (`https?://\S+`, `www.\S+`) with 「リンク」/"link";
      replace bare absolute paths (`(?<!\w)/(?:[\w.\-]+/)+[\w.\-]+`) with
      their basename (narration readability, DESIGN §5.3).
   5. Collapse whitespace runs; trim.
   6. Truncate per Req 1 (sentence boundary 。．！？!?. preferred within the
      last 30 %, else word boundary).
3. Documented non-goal (code comment cross-referencing DESIGN §9.2 T3): no
   persuasive/imperative-content detection here; the cross-component proof
   that approval prompts cannot be forged by agent text lives in issue 37's
   T3-e2e suite.
4. Performance: O(n); 20 KB input < 5 ms (perf test).
5. Corpus file `Tests/Fixtures/sanitizer-corpus.json` (input/expected
   pairs, ≥ 25): SSML smuggling incl. a `<speak>` tag with a 1 000-char
   attribute, bidi spoofing, zero-width splitting (「承\u{200B}認」 → plain
   「承認」 as content), markdown bombs, 10 000-char URL, mixed ja/en, emoji
   preservation (emoji are allowed), agent text that *imitates an approval
   prompt* ("say confirm 4 9") — passes through as inert prose (documented
   expectation: architectural defenses handle it).

## Acceptance Criteria

- [ ] Every pipeline step covered + full-pipeline corpus green.
- [ ] Long-attribute tag (>200 chars) removed; unclosed-`<` beyond 4096 kept
      literal (both tested).
- [ ] Truncation code-point-safe for CJK/emoji; suffix included within
      `maxLength`; cases at max, max±1, no-boundary.
- [ ] Perf < 5 ms / 20 KB.
- [ ] Pure, deterministic, Foundation-only imports.

## Validation

`swift test --filter SpeechTextSanitizerTests`.

## Dependencies

01.

## Non-goals

Semantic/imperative content filtering (see Req 3), profanity filtering, HUD
`detail` rendering (31 strips control chars only), secret redaction (04's
`Redactor` — different layer, applied at logs/transcripts).

## Design References

DESIGN.md §9.5, §9.2 (T3, amended), §5.3, §5.4.
