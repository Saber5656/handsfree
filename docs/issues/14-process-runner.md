# Title

ProcessRunner: child process spawn/stream/cancel/timeout with process groups

## Summary

Implement `HandsfreeAgent/Adapter/ProcessRunner`: a reusable async child
process utility with its own process group, line-oriented stdout streaming,
stderr ring buffer, SIGINT→SIGKILL cancellation, and timeouts.

## Context

`codex exec` turns are long-lived children that may spawn their own
descendants; cancellation must reliably terminate the whole tree
(DESIGN §6.2). Foundation's `Process` cannot set a process group, so this
component wraps `posix_spawn` directly.

## Scope

- One utility type + tests. No codex semantics (18), no JSONL (15).

## Detailed Requirements

1. API:
   ```swift
   struct ProcessSpec {
       let executable: String            // absolute path
       let arguments: [String]
       let workingDirectory: URL
       let environment: [String: String] // computed by caller
       let timeout: Duration?
   }
   final class ProcessRunner {
       func run(_ spec: ProcessSpec) throws -> RunningProcess
   }
   final class RunningProcess {
       let stdoutLines: AsyncThrowingStream<String, Error>  // newline-delimited, UTF-8
       var stderrTail: String { get }                       // ring buffer, 64 KB
       func cancel() async                                  // SIGINT pgid, 5 s grace, SIGKILL pgid
       func waitForExit() async -> ProcessExit              // .exited(code) | .signalled(Int32)
       let pid: pid_t
       let startedAt: Date
   }
   ```
2. Spawn via `posix_spawn` with `POSIX_SPAWN_SETPGROUP` (pgid = child pid).
   stdin: open `/dev/null` read-only (research doc: codex reads stdin when
   piped — must be closed/null). stdout/stderr: pipes with O_NONBLOCK reads on
   a dedicated reader task.
3. Line framing: split on `\n`; **cap a single line at 1 MB** — longer lines
   are truncated with a logged warning and delivered with a
   `truncated` marker suffix constant (DESIGN §9.5). Handle final unterminated
   line at EOF.
4. Cancellation: `kill(-pgid, SIGINT)`; if not exited within 5 s,
   `kill(-pgid, SIGKILL)`. Idempotent; safe after exit (ESRCH ignored).
   `stdoutLines` finishes (not throws) on cancellation-caused exit.
5. Timeout: if `spec.timeout` elapses (TestClock-injectable), behave exactly
   like `cancel()` but `waitForExit` reports the underlying exit AND a
   `timedOut` flag on `RunningProcess`.
6. Environment policy helper (used by 18):
   `LoginEnvironment.capture()` — the user's login-shell environment captured
   once per app run (same mechanism as 13), with **no Handsfree-added
   variables** (DESIGN §9.4).
7. Zombie prevention: always `waitpid` the child (dedicated wait task);
   `RunningProcess` deinit without wait logs an assertion in debug.
8. Tests use committed fixture scripts under `Tests/Fixtures/procs/`:
   `emit-lines.sh`, `sleep-forever.sh` (spawns a grandchild sleeper to verify
   group kill), `stderr-noise.sh`, `long-line.sh`, `exit-42.sh`.

## Acceptance Criteria

- [ ] Group kill: cancelling `sleep-forever.sh` also kills its grandchild
      (verified via `kill(grandchildPid, 0)` == ESRCH after cancel).
- [ ] SIGINT-ignoring child (fixture traps INT) is SIGKILLed after the 5 s
      grace (TestClock-compressed).
- [ ] 1 MB line cap: `long-line.sh` output delivered truncated with marker;
      stream continues afterwards.
- [ ] stderr ring buffer keeps only the last 64 KB.
- [ ] Timeout path sets `timedOut` and terminates the tree.
- [ ] Exit code and signal reporting correct (`exit-42.sh`, SIGKILL case).
- [ ] No zombies after 100 sequential runs (asserted via `waitpid` bookkeeping).

## Validation

`swift test --filter ProcessRunnerTests` (all local, no `live` tag needed).

## Dependencies

12 (module placement only).

## Non-goals

PTY allocation, shell interpretation (argv only — `sh -c` is forbidden by
DESIGN §9.5), Windows/Linux portability.

## Design References

DESIGN.md §6.2, §9.4, §9.5, §12; research doc (stdin quirk).
