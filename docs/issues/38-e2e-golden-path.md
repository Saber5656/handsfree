# Title

E2E golden-path suite and performance budget checks

## Summary

Promote the orchestrator E2E harness (28) into the product's regression gate:
complete the failure-path matrix, assert the ja/en golden-path *spoken
transcripts* against golden files, and add automated performance budget
checks.

## Context

ISSUE_PLAN §6 makes this suite the definition of "the loop works" in CI
(real audio/codex remain manual per QA checklist, issue 40). DESIGN Appendix A
scripts are the product's acceptance scenario (§1.4).

## Scope

- Test-only changes (harness utilities, golden files, perf checks). Product
  fixes discovered here follow the 37 rule (small inline / else follow-up).

## Detailed Requirements

1. Golden spoken-transcript tests: for A.1 (ja) and A.2 (en), the full ordered
   list of (earcon, spoken text) pairs recorded by MockTTS/arbiter spy is
   compared to committed golden files
   (`Tests/Fixtures/golden/spoken-ja.txt`, `spoken-en.txt`) — template
   changes update goldens consciously in review. Nonce digits are normalized
   (`{d1} {d2}` placeholders) before comparison.
2. Failure-path matrix (extends 28's ten scenarios; each asserts spoken error
   keys, task end-state, transcript records):
   preflight-fail dispatch; spawn-fail (bad codexPath mid-session);
   `hang`→timeout; `crash`; `malformed`→corrupted-stream failure;
   STT stream error mid-listening (mock throws) → error_recoverable →
   listening; TTS error (mock) → session continues; endpoint abandon
   (no speech 10 s) path; config-reload mid-session (verbosity change applies
   next narration); interrupted-task resume flow after simulated relaunch
   (new orchestrator over the same stores → pending queue offers resume →
   `codex exec resume` argv asserted).
3. Performance budget checks (DESIGN §15, logic-level):
   - intent decision ≤ 300 ms wall-clock with real matcher on 200-char input
     (already in 19; re-assert integrated);
   - narration event→HUD line emission ≤ 100 ms through the real pipeline
     (instrumented spy timestamps);
   - FSM reduce ≤ 1 ms p99 over the property-test corpus.
   Hardware-bound budgets (hotkey→listening 500 ms; idle footprint) get a
   measurement helper + a documented manual procedure appended to
   `docs/QA_CHECKLIST.md` (created in 40 — if 40 unmerged, create the file
   with only this section).
4. Flakiness policy: suite must pass 20 consecutive local runs
   (`for i in $(seq 20); do swift test --filter GoldenPathTests || exit 1; done`
   — evidence in PR); any sleep-based waits are replaced by clock injection.
5. CI wiring: suite runs in the standard test job (02); runtime budget ≤ 3 min
   with `FAKE_CODEX_SPEED=0`.

## Acceptance Criteria

- [ ] Golden ja/en transcripts committed and matched (with nonce
      normalization); a deliberate template change demo shows the diff review
      flow in the PR description.
- [ ] All listed failure paths tested (≥ 10 new scenarios) with end-state
      assertions.
- [ ] Perf checks green in CI; manual-measurement section added to the QA doc.
- [ ] 20-run stability evidence.
- [ ] Total suite runtime in CI ≤ 3 min.

## Validation

`swift test --filter GoldenPathTests E2ETests PerfBudgetTests` in CI; the
20-run loop output in the PR.

## Dependencies

28 (uses 17; touches docs owned by 40).

## Non-goals

Live-audio automation (impossible on CI runners), UI snapshot testing,
load/soak testing.

## Design References

DESIGN.md §1.4, Appendix A, §11, §12, §15; ISSUE_PLAN §6.
