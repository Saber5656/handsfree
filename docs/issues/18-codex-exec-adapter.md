# Title

CodexExecAdapter: turn execution with tier flags, resume, and cancellation

## Summary

Implement the production `AgentAdapter` composing preflight (13), process
runner (14), JSONL decoder (15), and response contract (16): exact argv
construction per risk tier, new/resume turns, cancellation, timeout, and
outcome assembly — integration-tested against FakeCodex (17).

## Context

This is the single execution path for all agent actions (ADR-002). Its argv
table IS the enforcement of the approval policy (DESIGN §6.5): a bug here is a
security bug (threat T7).

## Scope

- `HandsfreeAgent/Codex/CodexExecAdapter.swift` + argv builder + tests.
- Not: approval decisions (21), task bookkeeping (26), prompt text (24).

## Detailed Requirements

1. Argv builder (pure function, exhaustively unit-tested):
   - Base (new turn):
     `exec --json -C <projectPath> --output-schema <schemaResourcePath>
     -o <tmpLastMessagePath> [-m <model>] <prompt>`
   - Resume: `exec resume <threadID> --json -C … --output-schema … -o …
     [-m …] <prompt>`
   - Tier flags (DESIGN §6.5):
     `.t0Read` → `-s read-only`;
     `.t1Workspace` → `-s workspace-write`;
     `.t2Network` → `-s workspace-write -c sandbox_workspace_write.network_access=true`;
     `.t3Full` → `-s danger-full-access`.
   - **Forbidden flags** (unit test asserts absence for every tier/turn
     combination): `--dangerously-bypass-approvals-and-sandbox`,
     `--ephemeral`, `--skip-git-repo-check`, `--dangerously-bypass-hook-trust`.
2. Gate: `startTurn` throws `AdapterError.preflightFailed(problems)` when the
   cached preflight has blocking problems (unsafe binary, unconfirmed path
   change, not authenticated). `versionUnsupported` is non-blocking (warn).
3. Temp files: schema is resolved from `Bundle.module` once; `-o` file in a
   per-turn temp dir (`FileManager.temporaryDirectory/handsfree-turn-<uuid>`),
   deleted after outcome assembly (best-effort, logged).
4. Event pipeline: `ProcessRunner.stdoutLines` → `CodexEventDecoder` →
   forward `.event`s into `RunningTurn.events`; capture the LAST
   `agent_message` text for the contract parser; on `turn.completed`, assemble
   `TurnOutcome` with `ResponseContractParser.parse(...)`, thread id, usage.
5. Failure assembly:
   - `turn.failed`/stream error item → `.failed(reason)`;
   - process exit ≠ 0 with no terminal event → `.failed("process exited N: " +
     stderrTail-derived reason)`;
   - > 10 consecutive `.malformed` lines → cancel process,
     `.failed("stream corrupted")`;
   - cancel() → `.cancelled`; timeout → `.timedOut` (both via 14 semantics).
   Exit-code semantics are treated as secondary evidence per the research doc;
   record observed codes in the PR and update the research doc's exit-code
   note (known unknown #3).
6. Working directory: `spec.workingDirectory = projectPath`; environment:
   `LoginEnvironment.capture()` unmodified (DESIGN §9.4).
7. Integration tests (against FakeCodex, `codexPath` injected): one test per
   scenario of issue 17 asserting: emitted `AgentEvent` sequence shape,
   `TurnOutcome`, argv recorded in the invocation log (exact tier flags,
   resume id, schema/-o paths), temp-file cleanup.

## Acceptance Criteria

- [ ] Argv table test covers all 4 tiers × new/resume (8 cases) byte-exact,
      plus forbidden-flag absence.
- [ ] All FakeCodex scenarios produce the specified outcomes (happy,
      needs-input+resume continuity, blocked→escalated-resume completes,
      blocked→non-escalated repeats, failed, garbage→fallback summary,
      malformed→corrupted-stream failure, hang→timeout, crash→failed,
      slow-drip full delivery).
- [ ] Preflight gate blocks dispatch with problems (test with fixture binary).
- [ ] Cancellation kills the process group and yields `.cancelled` terminal
      outcome exactly once.
- [ ] A `live`-tagged smoke (real codex, `-s read-only`, trivial prompt in a
      scratch git repo) passes on the dev machine; observed exit codes noted.

## Validation

`swift test --filter CodexExecAdapterTests` (FakeCodex-based, CI-safe);
`live` smoke output in PR.

## Dependencies

13, 14, 15, 16, 17.

## Non-goals

Approval UX, tier *selection* (orchestrator/approval engine decide; adapter
only executes the requested tier), multi-turn planning.

## Design References

DESIGN.md §6.2, §6.5, §9.2 (T7), §12, §16 (#3); ADR-002, ADR-004; research doc.
