# Review resolution addendum

- Repository: `Saber5656/handsfree`
- Pull request: #1
- Original PR head before this resolution addendum: `9876834aab66b2fb0a075b42b4eac74b811c9d18`
- Scope: the review findings listed below are converted into normative design contracts and focused verification gates.
- The immutable current PR head is supplied by the parent task's fresh GitHub read immediately before review/reply/resolve; any later head change invalidates this review evidence and requires a fresh review.
- This addendum records design-level handling only; it does not claim implementation, test, build, CI, or security validation is complete.
- Per task instruction, the PR review bot is not re-triggered after these responses/resolutions.

## 1. Thread `PRRT_kwDOTNkGeM6QHQsl` — Run the build before querying the bin path

**Normative resolution**: The packaging path must invoke `swift build -c release --product HandsfreeApp` and only then resolve/copy the product from the reported build directory; `--show-bin-path` is informational and never substitutes for the build.

**Focused verification gate**: From a clean checkout, run the exact packaging command and assert that the product exists only after the explicit release build; fail if the path query alone is treated as success.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 2. Thread `PRRT_kwDOTNkGeM6QHQsm` — Use a regex that excludes live test suites

**Normative resolution**: The CI exclusion pattern must match the suite component before SwiftPM's method separator, using the canonical `LiveTests(?:\.|/)` form (or an equivalent tested form), and the same pattern must be used for every live suite.

**Focused verification gate**: Test identifiers with suite-only, dot-separated, slash-separated, and ordinary non-live cases; assert live hardware/Codex tests are excluded while unit tests remain selected.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 3. Thread `PRRT_kwDOTNkGeM6QHQsn` — Make the malformed fixture hit the corruption threshold

**Normative resolution**: The `malformed` FakeCodex scenario must emit at least 11 consecutive malformed lines, matching the `> 10` threshold, and the expected terminal result is exactly `.failed("stream corrupted")`.

**Focused verification gate**: Run the fixture through the adapter and assert the consecutive counter crosses the threshold, exactly one terminal failure is emitted, and a shorter malformed sequence does not falsely reach that oracle.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 4. Thread `PRRT_kwDOTNkGeM6QHQso` — Exclude docs from repo-wide security tripwires

**Normative resolution**: Security tripwires that prohibit product-code APIs are scoped to executable/product paths such as `Sources/` and security scripts; documentation is checked by a separate documentation-aware scan so intentional design references do not make the product-code gate fail.

**Focused verification gate**: Run the security script against the committed tree containing the documented terms and assert product-code violations fail while approved documentation references do not; separately verify the docs scan still detects accidental secrets or unsafe executable snippets.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 5. Thread `PRRT_kwDOTNkGeM6QHQsq` — Expose the dropped-line count

**Normative resolution**: `RunningProcess` must expose a public, deterministic `droppedLineCount` (or an explicitly equivalent result field) that increments for every discarded oversized line and is consumed by the adapter's stream-corruption decision.

**Focused verification gate**: Feed an oversized stdout line and a normal line, assert the count and emitted event are observable through the public API, and assert the adapter maps a dropped line to the documented corruption outcome.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 6. Thread `PRRT_kwDOTNkGeM6QHQsr` — Handle Japanese task-number suffixes

**Normative resolution**: `task_select` parsing must accept the documented Japanese numeric suffix forms, including `1番`, while preserving existing Arabic, digit-word, and ambiguity rules; the normalized selection must be the same task number.

**Focused verification gate**: Table-test `1`, `1番`, Japanese digit words plus `番`, invalid suffixes, and multiple pending tasks; assert only valid forms bind and ambiguity remains explicit.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 7. Thread `PRRT_kwDOTNkGeM6QHQss` — Redact multi-line keys before truncating

**Normative resolution**: Secret redaction runs before the 64 KiB retention cap and also treats an unmatched private-key begin marker as sensitive, replacing retained material with a safe placeholder before any diagnostic snapshot is persisted.

**Focused verification gate**: Use a PEM whose begin marker is inside the cap and end marker beyond it; assert no key material survives in logs/snapshots, and verify ordinary truncation still obeys the size limit.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 8. Thread `PRRT_kwDOTNkGeM6QHQsu` — Keep a production path for FakeCodex demo env

**Normative resolution**: Provide an explicit, narrowly scoped demo-mode path for the bundled FakeCodex localization scenario (`demo-ja`/`demo-en`), authorized only for the demo adapter and never as arbitrary production environment injection; alternatively the production demo must use the specified prompt-prefix mechanism. The chosen path must be documented consistently with issue 34.

**Focused verification gate**: Run the onboarding demo in both locales and assert the selected scenario is visible; run a normal production session and assert unapproved environment variables cannot alter provider behavior.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 9. Thread `PRRT_kwDOTNkGeM6QHQsv` — Do not read Core resources via App Bundle.module

**Normative resolution**: Resource ownership remains in `HandsfreeCore`; expose a Core-owned resource accessor/bundle helper for the smoke hook, or give the App target an explicitly owned smoke resource. `HandsfreeApp` must not call a synthesized `Bundle.module` accessor for another target.

**Focused verification gate**: Build both targets from a clean checkout and run the resource smoke; assert compilation succeeds and the phrases are loaded from the owning Core bundle.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 10. Thread `PRRT_kwDOTNkGeM6QHQsw` — Resolve the FSM speech-type import conflict

**Normative resolution**: The FSM contract must either import the Core speech types explicitly or define local effect enums and map them in the orchestrator; the document must not simultaneously forbid the required module and require its `SpeechPriority`/`Earcon` types.

**Focused verification gate**: Compile the FSM and orchestrator together, assert the mapping covers every effect and priority, and run a test proving no forbidden cross-target import remains.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 11. Thread `PRRT_kwDOTNkGeM6QHQsy` — Use one regex that excludes live test suites

**Normative resolution**: This duplicate live-suite requirement is normalized to the same shared `LiveTests(?:\.|/)` exclusion pattern as the CI contract above; no second divergent pattern is permitted.

**Focused verification gate**: Run the shared pattern against the same SwiftPM identifier matrix and assert both issue references select identical live/non-live sets.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 12. Thread `PRRT_kwDOTNkGeM6QHQsz` — Return a canonical hotkey binding

**Normative resolution**: Validation and normalization are separate observable operations: `HotkeyBinding.normalized()` returns a canonical immutable binding, while `validate()` checks the canonical form and reports errors. ConfigStore/Settings must store the normalized result rather than silently retaining non-canonical input.

**Focused verification gate**: Provide mixed-case and reordered modifiers, assert normalization is deterministic and idempotent, validation accepts the canonical result, and invalid keys remain rejected.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 13. Thread `PRRT_kwDOTNkGeM6QHQs1` — Emit a terminal outcome on clean EOF

**Normative resolution**: A child exit with code 0 but no wire-level `turn.completed` or `turn.failed` event is a protocol failure; the adapter must emit exactly one terminal `turnEnded` failure and never leave the caller waiting.

**Focused verification gate**: Exercise clean EOF with no terminal event, non-zero EOF, and a valid terminal event; assert one-and-only-one terminal outcome and the documented error mapping.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 14. Thread `PRRT_kwDOTNkGeM6QHQs2` — Make transcript opt-out crash-safe

**Normative resolution**: When transcripts are disabled or retention is zero, transcript/session records remain memory-only and no pathname containing sensitive text is created. Persisted mode uses the normal durable path; opt-out mode must not rely on end-of-session cleanup.

**Focused verification gate**: Force-kill/crash the process in each opt-out mode and inspect the filesystem after restart; assert no transcript temp file or recoverable sensitive content exists.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 15. Thread `PRRT_kwDOTNkGeM6QHQs4` — Reconstruct notification identifiers after relaunch

**Normative resolution**: Delivered notification request identifiers must be persisted with the task index or reconstructed by querying delivered notifications using the stable `threadIdentifier`; cleanup after relaunch must not depend on an in-memory-only list.

**Focused verification gate**: Deliver notifications, terminate/relaunch before acknowledgement, invoke cleanup, and assert all task notifications are removed without touching another task's thread.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 16. Thread `PRRT_kwDOTNkGeM6QHQs5` — Require numbered choice for multiple pending tasks

**Normative resolution**: `yes` is an implicit confirmation only when exactly one pending task exists. When multiple tasks are enumerated, the parser must require `task_select(N)`/the documented numbered form or a fresh-task command and must not bind the first task implicitly.

**Focused verification gate**: Test zero, one, and multiple pending tasks with `yes`, `1番`, and explicit selection; assert ambiguous confirmation never resumes the wrong task.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 17. Thread `PRRT_kwDOTNkGeM6QHQs7` — Bind terminal task rows instead of acknowledging them

**Normative resolution**: Selecting a completed/failed task from the menu first follows the TaskManager bind/display flow so a follow-up remains possible; acknowledgement occurs only after the terminal result is presented or the user explicitly acknowledges it.

**Focused verification gate**: Select completed and failed rows, assert they become bound/displayable and retain follow-up eligibility, then assert explicit acknowledgement makes them non-bindable.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 18. Thread `PRRT_kwDOTNkGeM6QHQs9` — Do not promise hotkey TTS interruption

**Normative resolution**: The v1 FAQ must state that hotkey interruption of speaking/narration is not guaranteed or is out of scope; it must not promise playback stopping unless the FSM, product issue, and tests add that behavior.

**Focused verification gate**: Review the FAQ against the FSM state table and run the documentation consistency check; assert no unsupported interruption promise remains.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.