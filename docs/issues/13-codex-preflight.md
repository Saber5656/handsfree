# Title

CodexPreflight: binary resolution with pinning, version and auth checks

## Summary

Implement `HandsfreeAgent/Preflight/CodexPreflight`: locate the codex binary
safely, pin it in config, verify version compatibility and login state, and
produce actionable `AgentPreflight` results for onboarding/Settings.

## Context

Threat T6 (DESIGN §9.2): a hijacked `codex_path` or PATH-shadowed binary would
run attacker code with the user's privileges on every task. Preflight is also
the UX front door — most first-run failures are "codex missing/not logged in"
(DESIGN §12).

## Scope

- Preflight logic + config pinning + typed problems. No UI (33/34 consume it).

## Detailed Requirements

1. Resolution order:
   1. `config.agent.codex_path` if set (must be absolute; else ignored with
      warning).
   2. Else `PATH` lookup of `codex` using the user's **login shell
      environment** (spawn `$SHELL -l -c 'command -v codex'` once; document
      why: GUI apps don't inherit shell PATH).
   3. On first successful resolution via PATH, **pin** the absolute path into
      `config.agent.codex_path` via ConfigStore (03).
2. Security checks on the resolved path (all must pass, else
   `PreflightProblem.unsafeBinary(reason)`):
   - regular file, executable bit set;
   - not world-writable; containing directory not world-writable;
   - owned by the current uid or root.
3. Change detection: if a previously pinned path no longer resolves or the
   file's (dev, inode) changed since pinning is NOT tracked in v1 — but if the
   *path value* changes (user edited config or re-pin needed), preflight
   reports `.binaryChanged(old:new:)` which the Settings UI (33) must surface
   as a confirmation; until confirmed, adapter refuses to run
   (`AgentPreflight.problems` non-empty ⇒ no dispatch). Store the confirmation
   as `agent.codex_path_confirmed: Bool` — add this key to the config schema
   in this PR (update DESIGN Appendix C.2 accordingly).
4. Version: run `<binary> --version` (ProcessRunner from 14 or a minimal local
   runner if 14 unmerged — coordinate; prefer depending on 14), parse
   `codex-cli X.Y.Z`. Tested range constant:
   `supported = 0.141.0 ..< 0.200.0` (from research doc); outside →
   `versionSupported=false` + problem `.untestedVersion` (warning-level:
   dispatch allowed, Settings shows notice — the range constant is updated by
   maintainers as versions are validated).
5. Auth: run `<binary> login status`; exit 0 → `.authenticated`; known
   "not logged in" output → `.notAuthenticated` (problem with fix hint
   "run `codex login` in a terminal"); anything else → `.unknown` (dispatch
   allowed; first real turn surfaces errors). Never invoke `codex login`
   itself (credentials are user-managed).
6. Caching: preflight result cached 5 min; `refresh()` forces re-run
   (Settings "Re-check" button).
7. Unit tests use fixture scripts (fake `codex` shell scripts emitting
   version/auth outputs, including a world-writable one created in a temp dir).

## Acceptance Criteria

- [ ] Resolution order + pinning behavior covered by tests (config empty →
      PATH pin; config set → used; relative path ignored).
- [ ] All unsafe-binary conditions individually tested.
- [ ] `.binaryChanged` flow: changing the pinned value without confirmation
      blocks dispatch (test through the adapter gate in 18 or a stub gate here).
- [ ] Version parse + range logic tested (in-range, below, above, garbage).
- [ ] Auth state mapping tested for the three outcomes via fixture scripts.
- [ ] `live` test with the real binary passes on the dev machine and reports
      authenticated.

## Validation

`swift test --filter CodexPreflightTests`; `live` output pasted in PR
(redacted as needed).

## Dependencies

03, 12 (types), 14 (process execution — see coordination note in Req 4).

## Non-goals

Running turns (18), managing codex config/login, multi-agent preflight.

## Design References

DESIGN.md §6.2, §9.2 (T6), §12; research doc (codex CLI 0.141.0).
