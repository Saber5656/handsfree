# Title

E2E golden-path suite and performance budget checks

## Summary

Promote the orchestrator E2E harness (28) into the product's regression
gate: golden spoken-transcript files for the ja/en golden paths, the
complete failure-path matrix, and automated performance checks — with an
explicit no-disabled-tests completion rule.

## Context

ISSUE_PLAN §6 makes this suite the CI definition of "the loop works"
(real audio/codex remain manual, issue 40's QA checklist). DESIGN Appendix A
is the acceptance scenario; DESIGN §15 budgets are split into
logic-measurable (here) and hardware-measurable (QA doc).

## Scope

- Test-only changes + golden files + a QA-doc section. Product bugs found:
  small fixes inline; larger ones become follow-up issues — but this issue
  CANNOT close with required golden/failure/perf tests failing or disabled.

## Detailed Requirements

1. Golden spoken transcripts: for Appendix A.1 (ja) and A.2 (en), the full
   ordered list of `(earcon?, spokenText)` pairs recorded by the MockTTS +
   arbiter spy is normalized (nonce digits → `{d1} {d2}`) and compared to
   `Tests/Fixtures/golden/spoken-ja.txt` / `spoken-en.txt`. Template changes
   update goldens consciously in review (the PR demonstrates the diff flow
   once with a deliberate template tweak, reverted).
2. Failure-path matrix (extends 28's ten scenarios; each asserts spoken
   error keys, task end-state, transcript records): preflight-fail
   dispatch; spawn-fail (bad codexPath mid-session); `hang`→timeout;
   `crash`; `malformed`→corrupted-stream failure; STT stream error
   mid-listening → error_recoverable → listening; TTS mock error → session
   continues; config-reload mid-session (verbosity change applies to the
   next turn's narration); interrupted-task resume after simulated relaunch
   (fresh orchestrator over the same stores → pending queue offers resume →
   `codex exec resume` argv asserted via FakeCodex log); dispatch → first
   narration under FakeCodex `slow-drip` (see perf below).
3. Performance checks (logic-level; wall-clock with mocked deps, no
   artificial delays, documented 3× CI tolerance):
   - utterance-final → intent decision ≤ 300 ms (integrated re-assertion of
     19/28);
   - agent event → orchestrator HUD-line emission ≤ 100 ms (measured at the
     orchestrator's HUD-line output — the rendered HUD belongs to 31 and is
     NOT in this measurement);
   - FakeCodex `happy` dispatch → first narration emission ≤ 1 s with
     `FAKE_CODEX_SPEED=0` (the real ≤ 3 s budget includes codex startup and
     is a QA-doc manual item);
   - FSM reduce p99 ≤ 1 ms over the property corpus.
   Hardware budgets (hotkey→listening 500 ms; idle CPU/RSS; in-session CPU;
   real dispatch→narration 3 s) get a measurement procedure appended to
   `docs/QA_CHECKLIST.md` (created here with only this section if 40 has
   not landed).
4. Stability: the full suite passes 20 consecutive local runs
   (`for i in $(seq 20); do swift test --filter
   'GoldenPathTests|E2ETests|PerfBudgetTests' || exit 1; done` — evidence in
   PR). No sleep-based waits (clock injection only).
5. CI: suite runs in the standard job (02) with `FAKE_CODEX_SPEED=0`;
   total added runtime ≤ 3 min.

## Acceptance Criteria

- [ ] Golden ja/en transcripts committed and matched (nonce-normalized);
      golden-update flow demonstrated once in the PR.
- [ ] All failure paths tested (≥ 10 new scenarios) with end-state
      assertions.
- [ ] Perf checks green in CI; QA-doc hardware-measurement section added.
- [ ] 20-run stability evidence over the full filter set.
- [ ] Suite runtime ≤ 3 min in CI (run link).

## Validation

`swift test --filter 'GoldenPathTests|E2ETests|PerfBudgetTests'` in CI; the
20-run loop output in the PR.

## Dependencies

28.
(Coordination notes, not prerequisites: the suite runs against the FakeCodex
fixture from 17, which 28 already requires; the hardware-measurement section
this issue adds to `docs/QA_CHECKLIST.md` is later folded into issue 40's
full QA document.)

## Non-goals

Live-audio automation, rendered-HUD latency (31), UI snapshot testing,
load/soak testing.

## Design References

DESIGN.md §1.4, Appendix A, §11, §12, §15; ISSUE_PLAN §6.
