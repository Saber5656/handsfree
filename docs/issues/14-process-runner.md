# Title

ProcessRunner: child process spawn/stream/cancel/timeout with process groups

## Summary

Implement `HandsfreeAgent/Adapter/ProcessRunner`: a reusable async child
process utility with its own process group, line-oriented stdout streaming,
stderr ring buffer, SIGINT→SIGKILL cancellation, timeouts, and the
`LoginEnvironment` capture helper.

## Context

`codex exec` turns are long-lived children that may spawn descendants;
cancellation must terminate the whole tree (DESIGN §6.2). Foundation's
`Process` cannot set a process group, so this wraps `posix_spawn`. DESIGN
§6.2 requires stdout lines over 1 MB to be **dropped** with a logged
warning (they are corrupt/hostile input, never delivered downstream).

## Scope

- `ProcessRunner`, `RunningProcess`, `LoginEnvironment`, fixtures, tests.
  No codex semantics (18), no JSONL (15).

## Detailed Requirements

1. API:
   ```swift
   public struct ProcessSpec {
       public let executable: String              // absolute path
       public let arguments: [String]
       public let workingDirectory: URL
       public let environment: [String: String]
       public let timeout: Duration?
   }
   public final class ProcessRunner {
       public init(clock: any Clock<Duration> = ContinuousClock())  // timeout + grace timing
       public func run(_ spec: ProcessSpec) throws -> RunningProcess
   }
   public enum ProcessExit: Equatable { case exited(Int32), signalled(Int32) }
   public final class RunningProcess {
       public let stdoutLines: AsyncThrowingStream<String, Error>
       public var stderrTail: String { get }       // ring buffer, last 64 KB
       public var timedOut: Bool { get }           // set when the timeout fired
       public let identity: ProcessIdentity        // pid, pgid, start tv_sec/tv_usec (issue 12 type)
       public func cancel() async
       public func waitForExit() async -> ProcessExit
   }
   ```
2. Spawn via `posix_spawn`: `posix_spawnattr_setflags(POSIX_SPAWN_SETPGROUP)`
   + `posix_spawnattr_setpgroup(&attr, 0)` — the child becomes its own group
   leader, so the returned child pid IS the pgid for `kill(-pid, …)`.
   Immediately after spawn, capture `proc_bsdinfo.pbi_start_tvsec/tvusec`
   via `proc_pidinfo` to fill `ProcessIdentity` (start-time source of truth
   for issue 26's recovery matching).
   stdin: `/dev/null` read-only (research: codex reads piped stdin).
   stdout/stderr: pipes drained by a dedicated reader task.
3. Line framing: split on `\n`; UTF-8 (invalid sequences replaced). **A
   single line exceeding 1 MB is dropped entirely** — the reader discards
   bytes until the next newline, increments `droppedLineCount` (public,
   consumed by 18's health check), and logs one warning per stream. Final
   unterminated line at EOF is delivered.
4. Cancellation: `kill(-pgid, SIGINT)`; if the process has not exited after
   a 5 s grace (runner clock), `kill(-pgid, SIGKILL)`. Idempotent; ESRCH
   ignored. `stdoutLines` finishes (not throws) on cancellation-caused exit.
5. Timeout: when `spec.timeout` elapses, behave exactly like `cancel()` and
   set `timedOut = true` before signalling.
6. `LoginEnvironment.capture()` (the documented DESIGN §9.5 exception):
   - Validates `$SHELL` is an absolute path to an executable regular file;
     otherwise falls back to `/bin/zsh`.
   - Runs `<shell> -l -c env` ONCE per app run (fixed literal command string,
     no interpolation); parses `KEY=VALUE` lines; caches the result.
   - Filter: removes every variable whose name starts with `HANDSFREE_`.
   - The captured environment is **in-memory only**: never logged, never
     written to transcripts/config/diagnostic snapshots (unit test with a
     spy sink; also listed in 37's audit).
7. Zombie prevention: dedicated `waitpid` task per child; debug assertion on
   deinit-without-wait.
8. Fixture contracts (`Tests/Fixtures/procs/`, each documented in a header
   comment): `emit-lines.sh` (N numbered lines then exit 0);
   `exit-42.sh`; `stderr-noise.sh` (>64 KB to stderr);
   `long-line.sh` (one 2 MB line, then `after` line);
   `sleep-forever.sh` (spawns a grandchild `sleep 1000`, writes both PIDs to
   a pidfile given as `$1`, then waits); `ignore-int.sh` (traps INT, writes
   pidfile, sleeps).

## Acceptance Criteria

- [ ] Group kill: cancelling `sleep-forever.sh` kills the grandchild too
      (pidfile → `kill(pid, 0)` returns ESRCH after cancel).
- [ ] `ignore-int.sh` is SIGKILLed after the grace (clock-driven test).
- [ ] `long-line.sh`: the 2 MB line is dropped (`droppedLineCount == 1`),
      the following `after` line IS delivered.
- [ ] stderr ring keeps only the last 64 KB of `stderr-noise.sh`.
- [ ] Timeout sets `timedOut` and terminates the tree.
- [ ] Exit reporting: `exit-42.sh` → `.exited(42)`; SIGKILL case →
      `.signalled(9)`.
- [ ] `identity.startTvSec/Usec` populated and stable across the process
      lifetime (compare against a second `proc_pidinfo` read).
- [ ] `LoginEnvironment`: `HANDSFREE_*` filtered; capture cached (single
      shell invocation across two calls); spy-sink test proves no logging of
      values; invalid `$SHELL` falls back to `/bin/zsh`.
- [ ] No zombies after 100 sequential runs.

## Validation

`swift test --filter 'ProcessRunnerTests|LoginEnvironmentTests'` (all local,
no live tag needed).

## Dependencies

01, 12 (`ProcessIdentity`).

## Non-goals

PTY allocation, shell interpretation of task commands (argv only — §9.5),
Windows/Linux portability.

## Design References

DESIGN.md §6.2, §9.4, §9.5 (exception), §12; research doc (stdin quirk);
issue 12 (`ProcessIdentity`).
