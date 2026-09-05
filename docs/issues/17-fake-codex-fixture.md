# Title

FakeCodex: scripted codex-CLI stand-in for tests, CI, and onboarding demo

## Summary

Build the `Sources/FakeCodex` executable (SwiftPM product `fake-codex`) that
mimics the `codex exec` CLI surface and emits scripted JSONL per scenario —
deterministic, offline — plus the golden scenario tests that pin its output.

## Context

CI has no codex login and must never call the network (DESIGN §11). Adapter
(18), orchestrator (28), golden-path (38), and onboarding-demo (34) tests run
against this fixture. Its CLI surface tracks the research-doc pin
(codex-cli 0.141.0). Scenario gates enforce the exact tier flags of DESIGN
§6.5 so escalation tests are meaningful (T7).

## Scope

- FakeCodex executable + built-in scenarios + `FakeCodexScenarioTests`
  (golden output tests ARE in scope — they define the fixture's contract).

## Detailed Requirements

1. CLI surface (arg-compatible subset): `fake-codex exec [flags] <prompt>` /
   `fake-codex exec resume <thread_id> [flags] <prompt>`; accepted flags:
   `--json`, `-C <dir>`, `-s <mode>`, `-c key=value` (repeatable),
   `--output-schema <file>`, `-o <file>`, `-m <model>`. Unknown flag →
   exit 2 + stderr message (drift guard).
2. Scenario selection: env `FAKE_CODEX_SCENARIO` (primary) or prompt prefix
   `@scenario:<name>` (demo mode). Global `FAKE_CODEX_SPEED` (float
   multiplier; `0` = no delays). Invocation recording: if `FAKE_CODEX_LOG`
   is set, append one JSON line per invocation: `{argv, cwd, scenario, ts}`.
3. Scenario wire contracts — every scenario has a committed canonical output
   template under `Sources/FakeCodex/Scenarios/<name>.jsonl.template`
   (placeholders only for thread id and timestamps), and the JSONL follows
   the issue-15 field table:
   - `happy`: `thread.started` (`fake-<uuid>`), `turn.started`, 2
     `command_execution` completions (exit_code 0), 1 `file_change`
     (summary, file_count 2), final `agent_message` whose text is a VALID
     contract JSON (`status=ok`, ja/en voice_summary chosen by
     `FAKE_CODEX_LANG`), `turn.completed` with fixed usage numbers; writes
     the same contract JSON to the `-o` file.
   - `needs-input`: like happy but final contract `status=needs_input` +
     `question`; `resume <id>` with any prompt → happy continuation echoing
     the SAME thread id.
   - `blocked-network`: final contract `status=blocked`,
     `blocked_reason=needs_network`, `blocked_action="push branch X to origin"`.
     On `resume`: completes (happy tail) **only if** argv contains BOTH
     `-s workspace-write` AND `-c sandbox_workspace_write.network_access=true`;
     otherwise emits the same blocked contract again.
   - `blocked-full`: `blocked_reason=needs_full_access`. On `resume`:
     completes **only if** argv contains `-s danger-full-access` (a network
     `-c` flag does NOT satisfy it); otherwise blocked again.
   - `failed`: `turn.failed` with reason.
   - `garbage`: valid events, then a final `agent_message` that is NOT
     contract JSON (prose + code fences) and no `-o` file (exercises the
     issue-16 fallback).
   - `malformed`: interleaves 3 broken JSON lines among valid events.
   - `hang`: `thread.started` + `turn.started`, then sleeps forever; must
     die on SIGINT (no trap).
   - `crash`: two events then exit 1 with no terminal event.
   - `slow-drip`: 20 `command_execution` completions at 200 ms × SPEED.
   - `demo-ja` / `demo-en`: friendly onboarding scripts (localized
     messages, no file changes), used by 34.
4. Thread ids: `happy` generates `fake-<uuid>`; `resume` echoes the given
   id. Exit codes: 0 for completed/blocked/needs-input scenarios, 1 for
   `crash`, 130 on SIGINT.
5. `FakeCodexScenarioTests` (in `HandsfreeAgentTests`): runs the built
   product (located via a TestSupport `productsDirectory` helper) for every
   scenario with `FAKE_CODEX_SPEED=0`, asserting stdout equals the canonical
   template (after placeholder substitution) and the invocation log records
   exact argv.

## Acceptance Criteria

- [ ] Every scenario's stdout matches its committed template (golden tests).
- [ ] Resume continuity: needs-input → resume → same thread id → completes.
- [ ] Escalation gates: blocked-network repeats without the network `-c`
      flag and completes with it; blocked-full REJECTS the network flag and
      completes only with `-s danger-full-access` (asserted via templates +
      invocation log).
- [ ] `hang` dies on SIGINT within 1 s; `crash` emits no terminal event;
      unknown flag → exit 2.
- [ ] Full scenario test suite runs < 5 s with `FAKE_CODEX_SPEED=0`.

## Validation

`swift test --filter FakeCodexScenarioTests`.

## Dependencies

12, 15 (field table is the wire contract source).

## Non-goals

Simulating real sandboxing/file edits (events only), auth flows, non-exec
subcommands, bundling into the app (34 amends make-app.sh).

## Design References

DESIGN.md §11, §6.2–6.5, §8.4; research doc (CLI surface); issue 15 (field
table); DESIGN §9.2 (T7).
