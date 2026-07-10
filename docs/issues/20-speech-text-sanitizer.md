# Title

SpeechTextSanitizer: the validation layer for everything spoken or notified

## Summary

Implement `HandsfreeCore/Dialogue/SpeechTextSanitizer`: the single sanitation
+ truncation utility applied to any agent-derived text before it reaches TTS,
notifications, or approval announcements.

## Context

Agent output is untrusted (threat T3, DESIGN §9.2/§9.5): `voice_summary`,
`question`, `blocked_action`, and narration payloads could carry control
characters, markup, homoglyph tricks, or social-engineering text. The
structural defenses (template-only approval prompts + earcon) live elsewhere;
this component guarantees the *content* is inert and speakable.

## Scope

- One pure utility + exhaustive tests. Consumers: 16 (caps), 21, 23, 35.

## Detailed Requirements

1. API:
   ```swift
   struct SpeechTextSanitizer {
       static func sanitize(_ s: String, maxLength: Int, locale: SpeechLocale) -> String
       static func truncate(_ s: String, max: Int) -> String  // code-point safe, word/sentence aware
   }
   ```
2. Sanitize pipeline (order matters; each step unit-tested):
   1. Unicode normalization NFC.
   2. Remove control characters (C0/C1 except `\n` → space), zero-width
      characters (ZWSP/ZWNJ/ZWJ/BOM), bidi control characters
      (U+202A–202E, U+2066–2069).
   3. Strip markup: HTML/XML-like tags `<[^>]{1,200}>` (incl. SSML-like
      `<speak>`, `<break>`), markdown emphasis/backticks/fences, `#` headers,
      list markers at line starts.
   4. Replace URLs (`https?://\S+`, `www.\S+`) with the locale word
      「リンク」/"link"; replace bare absolute paths (`(?<!\w)/(?:[\w.\-]+/)+
      [\w.\-]+`) with their last path component (basename) — narration
      readability (DESIGN §5.3).
   5. Collapse whitespace runs; trim.
   6. Truncate to `maxLength` preferring sentence boundary (。．！？!?.)
      within the last 30%, else word boundary, appending 「…以下省略」/"…
      truncated" locale suffix when cut.
3. Explicit non-goal documented in code: the sanitizer does NOT attempt to
   detect persuasive/imperative content ("say confirm 4 9") — that defense is
   architectural (nonce unpredictability + template-only announcements +
   earcon; DESIGN §5.4). A doc-comment cross-references the threat model so
   future contributors don't add naive keyword filters here.
4. Performance: O(n); 20 KB input sanitized < 5 ms (perf test).
5. Injection corpus file `Tests/Fixtures/sanitizer-corpus.json`: pairs of
   input/expected covering: SSML smuggling, bidi spoofing, zero-width
   splitting of the approve keyword (「承\u{200B}認」 must come out as 「承認」
   *as plain content* — and a companion test in 19 proves matcher input is
   never sourced from agent text), markdown bombs, 10 000-char URL, mixed
   ja/en, emoji preservation (emoji are allowed content).

## Acceptance Criteria

- [ ] Every pipeline step covered by focused tests + full-pipeline corpus
      (≥ 25 cases) green.
- [ ] Truncation is code-point safe for CJK/emoji and sentence-aware (cases
      at exactly max, max±1, long-no-boundary).
- [ ] Perf test < 5 ms / 20 KB.
- [ ] Pure function: no locale/global state; deterministic outputs.

## Validation

`swift test --filter SpeechTextSanitizerTests`.

## Dependencies

01.

## Non-goals

Semantic content filtering, profanity filtering, markdown *rendering* for the
HUD (HUD shows `detail` in a scrollable text view as-is minus control chars —
HUD-side handling lives in 31).

## Design References

DESIGN.md §9.5 (boundary table), §9.2 (T3), §5.3, §5.4.
