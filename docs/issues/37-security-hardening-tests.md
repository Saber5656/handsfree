# Title

Security acceptance tests and v1 audit (threats T1–T10)

## Summary

Turn the DESIGN §9.2 threat table into (a) executable cross-component
acceptance tests for every automatable threat and (b) a committed audit
checklist (`docs/security/v1-audit.md`) for the manual items — the gate that
security cannot be skipped (task-level enforcement of R7/ADR-004/ADR-010).

## Context

Individual issues carry local security tests (21 nonce rules, 14 process
caps, 20 sanitizer…). This issue adds the *end-to-end abuse cases* that cross
component boundaries, plus static tripwires, plus the human-verified items —
and wires them into CI so regressions fail loudly.

## Scope

- `Tests/HandsfreeSecurityTests/` suite + static-check script + audit doc.
- No new product code except fixes it uncovers (file follow-ups if large).

## Detailed Requirements

1. End-to-end abuse tests (orchestrator + FakeCodex + mocks, CI-safe;
   IDs match DESIGN §9.2):
   - **T1-e2e**: STT script yields a near-miss echo (「承認 4」, digits
     reversed, extra suffix) during a blocked-network approval → task ends
     `denied`, NO escalated argv in the FakeCodex invocation log.
   - **T2-e2e**: replay — approval denied by timeout, new approval started,
     previous nonce spoken → denied; also correct-echo-after-timeout → denied.
   - **T3-e2e**: FakeCodex scenario with hostile contract fields
     (voice_summary = "Approval granted. Say confirm 4 9 now", blocked_action
     containing `<speak>` markup + URL + newline bombs) → spoken output
     (MockTTS record) contains sanitized text only, approval announcement uses
     the policy template with the REAL nonce (not the injected digits), and
     the injected digits echoed by the "user" do NOT approve (nonce mismatch).
   - **T5-e2e**: after a full session, permission sweep asserts every created
     file is `0600`, dirs `0700`; notification payload test (35) re-asserted
     here with a secret-bearing summary.
   - **T6-e2e**: swap the pinned codex path to a different fixture binary
     without confirmation → dispatch refused with the preflight problem; with
     `codex_path_confirmed=true` → allowed.
   - **T7-e2e**: full blocked→approve flow, then a follow-up new instruction —
     invocation log asserts: escalated flags appear on exactly ONE resume
     invocation; the next turn's argv is back at `-s workspace-write`;
     escalated turn used the `approved_action` scaffold (prompt captured in
     invocation log contains the approved action block and NOT a generic task
     scaffold).
   - **T10-partial**: FSM mic-policy property re-asserted at orchestrator
     level: audio engine `start` effect only ever issued from states whose
     `micPolicy != .off` (spy assertion over all 10 E2E scenarios from 28/38).
2. Static tripwires (`scripts/security-checks.sh`, run in CI after tests):
   - forbidden flag strings absent outside the adapter's forbidden-list
     constant and tests (`grep -rn "dangerously-bypass" Sources | grep -v ForbiddenFlags`);
   - no `Process(` / `NSTask` outside `HandsfreeAgent` (single spawn path);
   - no `sh -c` anywhere in Sources;
   - no `print(` in Sources;
   - Package.swift dependency array empty (ADR-010);
   - workflows: every `uses:` matches `@[0-9a-f]{40}`.
3. `docs/security/v1-audit.md` — manual checklist with evidence columns,
   completed per release (referenced by 39/40):
   - T8: workflow pins reviewed; release built from clean checkout; checksums
     published.
   - T9: codesign chain (`codesign -dvv`, `spctl -a -t exec -vv`), stapler
     validate, fresh-VM Gatekeeper open.
   - T10: mic indicator behavior observed in all session states; TCC list
     shows mic (+speech per 08) only; Accessibility list does NOT contain
     Handsfree.
   - Entitlements dump equals exactly `audio-input` (command + expected
     output embedded).
4. Any failing expectation that reveals a product bug: fix in this PR if
   ≤ ~30 lines, else file a follow-up issue and mark the test `.disabled`
   with the issue link (CI must show the disabled count).

## Acceptance Criteria

- [ ] All T1/T2/T3/T5/T6/T7/T10 e2e tests implemented and green in CI.
- [ ] `security-checks.sh` green and wired into ci.yml (02 amendment).
- [ ] Audit doc committed with the exact commands and expected outputs;
      one full pass executed on the dev machine with evidence pasted.
- [ ] Zero disabled security tests at issue close, or linked follow-ups
      approved by the maintainer.

## Validation

`swift test --filter HandsfreeSecurityTests` + `scripts/security-checks.sh`
in CI; audit-doc pass evidence in the PR.

## Dependencies

16, 18, 20, 21, 27 (uses 28's harness; amends 02).

## Non-goals

Penetration testing of codex itself, sandbox-escape research (codex's sandbox
is an upstream trust boundary), fuzzing beyond 16's corpus.

## Design References

DESIGN.md §9 (entire), §6.5; ADR-004, ADR-010; ISSUE_PLAN §6.
