# Title

CodexPreflight: binary resolution with pinning, version and auth checks

## Summary

Implement `HandsfreeAgent/Preflight/CodexPreflight`: locate the codex binary
safely, pin and confirm it in config, verify version compatibility and login
state, and produce `AgentPreflight` results (blocking problems vs
non-blocking warnings) for the adapter gate and Settings.

## Context

Threat T6 (DESIGN §9.2): a hijacked `codex_path` or PATH-shadowed binary
would run attacker code on every task. Confirmation state is stored as
`agent.codex_path_confirmed_path` (the exact path the user last trusted —
DESIGN Appendix C.2), so any path change is detectable. Preflight is also the
UX front door for "codex missing / not logged in" (DESIGN §12).

## Scope

- Preflight logic + typed problems/warnings + config pinning. No UI (33/34).

## Detailed Requirements

1. Resolution order:
   1. `config.agent.codex_path` if set (already path-validated by 03).
   2. Else PATH lookup: capture the login environment once via issue 14's
      `LoginEnvironment.capture()` (the documented §9.5 exception), then
      search the captured `PATH` entries **in Swift** (join + executable
      check per entry — no shell interpretation of the lookup itself).
   3. On first successful resolution, pin the absolute path into
      `agent.codex_path` AND set `agent.codex_path_confirmed_path` to the
      same value (initial trust = the moment of first setup) via ConfigStore.
2. Security checks on the resolved path (violation ⇒
   `PreflightProblem.unsafeBinary(reason:)`, blocking): regular file;
   executable; not world-writable; parent dir not world-writable; owned by
   current uid or root. (These re-run at every preflight even though 03
   validates on config load — defense in depth.)
3. Change detection: if resolved path ≠ `codex_path_confirmed_path` →
   `PreflightProblem.binaryChanged(old: String?, new: String)` (blocking).
   Settings (33) clears it by setting `codex_path_confirmed_path` to the new
   path after explicit user confirmation. Preflight itself NEVER updates the
   confirmed path except during the initial pinning of Req 1.3.
4. Version: run `<binary> --version` via issue 14's `ProcessRunner`
   (5 s timeout); parse `codex-cli X.Y.Z`. Tested range constant
   `0.141.0 ..< 0.200.0` (research doc). Outside range or unparseable →
   `PreflightWarning.untestedVersion(found: String?)` (non-blocking; Settings
   shows a notice). Maintainers update the constant as versions are
   validated.
5. Auth: run `<binary> login status` (5 s timeout). Oracle (pinned from the
   real binary, 2026-07-08, codex-cli 0.141.0: stdout
   `"Logged in using ChatGPT"`, exit 0):
   - exit 0 → `.authenticated`;
   - exit ≠ 0 AND (stderr+stdout) matches `(?i)not logged in` →
     `.notAuthenticated` + `PreflightProblem.notAuthenticated` (blocking,
     fix hint: run `codex login` in a terminal);
   - anything else → `.unknown` (no problem entry; the first real turn
     surfaces errors).
   Fixture scripts for all three cases live in `Tests/Fixtures/preflight/`.
   Never invoke `codex login` itself (credentials are user-managed).
6. Gate semantics (consumed by 18): `problems.isEmpty == false` ⇒ dispatch
   refused. `warnings` never block. Add `PreflightProblem` cases:
   `.binaryNotFound`, `.unsafeBinary(reason: String)`,
   `.binaryChanged(old: String?, new: String)`, `.notAuthenticated`;
   `PreflightWarning` cases: `.untestedVersion(found: String?)`,
   `.authUnknown`.
7. Caching: result cached 5 min (injected clock); `refresh()` forces re-run.
8. All tests use fixture scripts (fake `codex` shell scripts under
   `Tests/Fixtures/preflight/`: version-ok, version-garbage, auth-ok,
   auth-not-logged-in, auth-weird-exit, plus a world-writable copy created
   in a temp dir at test time).

## Acceptance Criteria

- [ ] Resolution order + initial pin/confirm behavior tested (config empty →
      PATH pin sets both keys; config set → used as-is).
- [ ] Each unsafe-binary condition individually tested.
- [ ] `binaryChanged` produced when resolved ≠ confirmed; cleared only by
      externally updating the confirmed path (simulating 33).
- [ ] Version parse/range: in-range, below, above, garbage → correct
      warning behavior.
- [ ] Auth mapping tested for all three outcomes via fixtures.
- [ ] Cache honors TTL and `refresh()` (TestClock).
- [ ] `CodexPreflightLiveTests` (manual): real binary on the dev machine
      reports authenticated + in-range version; output pasted in PR.

## Validation

`swift test --filter CodexPreflightTests` (CI-safe);
`swift test --filter CodexPreflightLiveTests` locally.

## Dependencies

03, 12, 14.

## Non-goals

Running turns (18), managing codex config/login, Settings UI (33),
multi-agent preflight.

## Design References

DESIGN.md §6.2, §9.2 (T6), §9.5 (env-capture exception), §12, Appendix C.2;
research doc (codex CLI 0.141.0; login-status oracle).
