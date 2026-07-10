# Title

FakeCodex: scripted codex-CLI stand-in for tests, CI, and onboarding demo

## Summary

Build a small executable (`Sources/FakeCodex`, SwiftPM executable target) that
mimics the `codex exec` CLI surface and emits scripted JSONL streams for every
scenario the adapter and E2E suites need — deterministic, offline, and also
used as the onboarding demo backend.

## Context

CI has no codex login and must never call the network (DESIGN §11). All
adapter (18), orchestrator (28), golden-path (38) and onboarding-demo (34)
tests run against this fixture. Its CLI surface must track the real one pinned
in the research doc (0.141.0).

## Scope

- FakeCodex executable + scenario scripts + docs. Not the tests themselves.

## Detailed Requirements

1. CLI surface (subset, arg-compatible with real codex so the adapter needs
   no test-specific branches):
   - `fake-codex exec [flags] <prompt>` and
     `fake-codex exec resume <thread_id> [flags] <prompt>`;
   - accepts and records (to a side-channel log file, see Req 4):
     `--json`, `-C <dir>`, `-s <mode>`, `-c key=value` (repeatable),
     `--output-schema <file>`, `-o <file>`, `-m <model>`; unknown flags →
     exit 2 with an error on stderr (so accidental flag drift fails loudly).
2. Scenario selection: env var `FAKE_CODEX_SCENARIO` (primary) or leading
   token in the prompt `@scenario:<name>` (for demo mode). Scenarios (each a
   built-in script; JSONL matches the research-doc schema):
   - `happy` — thread.started, turn.started, 2 command items, file_change,
     final agent_message = valid contract JSON (`status=ok`), turn.completed;
     writes `-o` file.
   - `needs-input` — final contract `status=needs_input` + question; on
     `resume` with any prompt → `happy` continuation with SAME thread id.
   - `blocked-network` / `blocked-full` — contract `status=blocked` with the
     matching `blocked_reason`/`blocked_action`; on resume with sandbox flag
     escalated (`-s` recorded value or network `-c` present) → `happy`; on
     resume WITHOUT escalation → `blocked-*` again (asserts tier mapping).
   - `failed` — turn.failed.
   - `garbage` — valid events then a non-contract final message (exercises
     fallback chain).
   - `malformed` — interleaves broken JSON lines.
   - `hang` — emits turn.started then sleeps forever (timeout/cancel tests);
     must die on SIGINT (default handler — do not trap).
   - `crash` — emits two events then exits 1 without terminal event.
   - `slow-drip` — 20 events at 200 ms intervals (narration throttle tests).
   - `demo-ja` / `demo-en` — friendly onboarding scripts (localized
     agent_message texts, no file changes), used by 34.
3. Timing: per-scenario line delays; global speed factor
   `FAKE_CODEX_SPEED` (e.g. `0` = no delays for CI).
4. Invocation recording: append one JSON line per invocation
   (argv, cwd, env-selected scenario, timestamp) to
   `$FAKE_CODEX_LOG` if set — adapter tests assert exact argv (tier flags,
   schema path, resume id) from this file.
5. Thread ids: `happy` generates `fake-<uuid>`; `resume` echoes the given id
   (continuity assertions in 18/26).
6. Build/packaging: normal SwiftPM executable product `fake-codex`; CI builds
   it with the package; tests locate it via
   `productsDirectory` helper in TestSupport.

## Acceptance Criteria

- [ ] Each scenario emits its documented event sequence (golden test per
      scenario runs the binary and asserts stdout lines).
- [ ] Resume continuity: `needs-input` → resume → same thread id, completes.
- [ ] Escalation gate: `blocked-network` repeats when re-run without the
      network `-c` flag and completes with it (both asserted via invocation log).
- [ ] `hang` dies on SIGINT within 1 s; `crash` produces no terminal event.
- [ ] Unknown flag → exit 2 (drift guard).
- [ ] `FAKE_CODEX_SPEED=0` makes the full scenario suite run < 5 s.

## Validation

`swift test --filter FakeCodexScenarioTests` (spawns the built product).

## Dependencies

12, 15 (schema/event shape source of truth).

## Non-goals

Simulating codex sandboxing/file edits (it only *pretends* via events), auth
flows, non-exec subcommands.

## Design References

DESIGN.md §11, §6.2–6.5, §8.4 (demo mode); research doc (CLI surface, schema).
