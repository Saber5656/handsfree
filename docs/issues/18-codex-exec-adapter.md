# Title

CodexExecAdapter: turn execution with tier flags, resume, and cancellation

## Summary

Implement the production `AgentAdapter` composing preflight (13), process
runner (14), wire decoder (15), and contract parser (16): canonical argv per
risk tier, new/resume turns, wire→domain event mapping, terminal outcome
assembly, cancellation, and timeout — integration-tested against FakeCodex
(17).

## Context

Single execution path for all agent actions (ADR-002). The argv table IS the
enforcement of the approval policy (DESIGN §6.5); a bug here is a security
bug (T7). Terminal semantics follow the amended DESIGN §6.1: exactly one
`turnEnded(TurnOutcome)` per turn.

## Scope

- `HandsfreeAgent/Codex/CodexExecAdapter.swift` + argv builder + wire-event
  mapper + tests. Not: approval decisions (21), task bookkeeping (26),
  prompt text (24).

## Detailed Requirements

1. Canonical argv (exhaustively unit-tested; exact arrays for all 8 cases —
   4 tiers × new/resume — written out in the tests):
   - New: `exec --json -C <projectPath> <tierFlags…> --output-schema
     <schemaPath> -o <lastMessagePath> [-m <model>] <prompt>`
   - Resume: `exec resume <threadID> --json -C <projectPath> <tierFlags…>
     --output-schema <schemaPath> -o <lastMessagePath> [-m <model>] <prompt>`
   - `<tierFlags…>`: `.t0Read` → `-s read-only`; `.t1Workspace` →
     `-s workspace-write`; `.t2Network` → `-s workspace-write -c
     sandbox_workspace_write.network_access=true`; `.t3Full` →
     `-s danger-full-access`.
   - **Forbidden flags** (single `ForbiddenFlags` constant; unit test asserts
     absence for every case): `--dangerously-bypass-approvals-and-sandbox`,
     `--dangerously-bypass-hook-trust`, `--ephemeral`,
     `--skip-git-repo-check`.
2. Preflight gate: `startTurn` throws `AdapterError.preflightFailed([PreflightProblem])`
   when the cached preflight has any (blocking) problem; warnings never
   block.
3. Environment: `LoginEnvironment.capture()` (already `HANDSFREE_*`-filtered
   by 14). Test seam: `init(…, extraEnvironment: [String: String] = [:])` —
   merged ONLY in tests to pass `FAKE_CODEX_SCENARIO/_LOG/_SPEED`; a unit
   test asserts production construction paths never populate it (grep-able
   constant + 37 static check).
4. Wire→domain mapping (from 15's `CodexWireEvent`):
   `threadStarted→.threadStarted`; `turnStarted→.turnStarted` (+ decoder
   state reset); `itemCompleted→.item(mapped AgentItem)` — command exit_code
   nil/0 → `.running`/`.succeeded`, nonzero → `.failed`; `errorItem` maps to
   `.item(.errorItem)` and is **never terminal** (codex quirk);
   `.ignored(.unknownEventType(first occurrence))` → one `Log.agent` warning.
5. Terminal assembly — exactly one `turnEnded(TurnOutcome)`, mapping table:
   | Condition | TurnStatus | contract |
   |---|---|---|
   | wire `turnCompleted` | `.completed` | `ResponseContractParser.parse(lastAgentMessageText, -o file)` result embedded (incl. fallback/ repairs) |
   | wire `turnFailed(reason)` or `streamError` | `.failed(reason)` | nil |
   | process exit ≠ 0 with no wire terminal | `.failed("process exited N: <stderr-derived reason>")` | nil |
   | > 10 consecutive `.malformed` lines OR `droppedLineCount > 0` at a malformed burst | cancel process → `.failed("stream corrupted")` | nil |
   | `cancel()` invoked | `.cancelled` | nil |
   | runner `timedOut` | `.timedOut` | nil |
   Observed real exit codes are recorded in the PR and appended to the
   research doc's exit-code note (known unknown #3).
6. Temp files: schema path resolved from `Bundle.module` once; `-o` file in
   `FileManager.temporaryDirectory/handsfree-turn-<uuid>/`; directory
   removed after outcome assembly (best-effort, logged).
7. `RunningTurn.processIdentity` populated from 14's `RunningProcess.identity`.
8. Integration tests vs FakeCodex (via `extraEnvironment`), one per issue-17
   scenario, asserting: emitted `AgentEvent` sequence shape, final
   `TurnOutcome` (status + contract fields + repairs), argv from the
   invocation log (exact tier flags, resume id, schema/-o paths), temp-dir
   cleanup, and for the escalation scenarios the full
   blocked→resume-escalated→completed / blocked→resume-unescalated→blocked
   matrix.

## Acceptance Criteria

- [ ] Argv table: 8 canonical arrays byte-exact + forbidden-flag absence.
- [ ] All FakeCodex scenarios produce the mapped outcomes (happy;
      needs-input + resume continuity; blocked-network/full escalation
      matrix incl. the network-flag-rejected-for-full case; failed; garbage
      → `.completed` with `source=.fallback`; malformed → `.failed("stream
      corrupted")`; hang → `.timedOut`; crash → `.failed(process exited…)`;
      slow-drip full delivery).
- [ ] Preflight gate blocks on problems; warnings don't block (stubbed
      preflight).
- [ ] `cancel()` → group killed, single `turnEnded(.cancelled)`.
- [ ] `CodexExecAdapterLiveTests` (manual): real codex, `-s read-only`,
      trivial prompt in a scratch git repo; observed exit codes noted +
      research doc updated.

## Validation

`swift test --filter CodexExecAdapterTests` (FakeCodex-based, CI-safe);
`swift test --filter CodexExecAdapterLiveTests` locally.

## Dependencies

13, 14, 15, 16, 17.

## Non-goals

Approval UX/tier selection (21/28 decide; adapter executes the requested
tier), multi-turn planning, task persistence (26).

## Design References

DESIGN.md §6.1 (amended), §6.2, §6.5, §9.2 (T7), §12, §16 (#3); ADR-002,
ADR-004; research doc.
