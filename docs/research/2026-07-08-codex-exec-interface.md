# Research: `codex exec` integration surface

- Date: 2026-07-08
- Verified against: codex-cli **0.141.0** installed on the development machine (`codex --version`)
- Method: `codex exec --help` capture + live probe run + official docs
- Status: facts pinned for DESIGN.md §Agent Adapter; re-verify on codex upgrades

## Why this matters

Handsfree v1 drives coding agents exclusively through the Codex CLI headless mode
(`codex exec`). The adapter contract in DESIGN.md must match the *actual* CLI surface,
not remembered or assumed flags. Everything below was captured from the real binary
or official documentation.

## CLI surface (from `codex exec --help`, 0.141.0)

Relevant options confirmed to exist:

| Flag | Meaning | Handsfree usage |
|---|---|---|
| `[PROMPT]` / stdin | Initial instructions | Task prompt built by PromptScaffold |
| `--json` | Print events to stdout as JSONL | Always on; primary event stream |
| `--output-schema <FILE>` | JSON Schema for the model's final response shape | Voice summary contract (see below) |
| `-o, --output-last-message <FILE>` | Write last agent message to file | Fallback capture of final message |
| `-s, --sandbox <MODE>` | `read-only` \| `workspace-write` \| `danger-full-access` | Risk tier mapping |
| `-C, --cd <DIR>` | Working root for the agent | Project registry path |
| `--add-dir <DIR>` | Extra writable directories | Not used in v1 (single-project tasks) |
| `-m, --model <MODEL>` | Model override | Optional per-project setting |
| `-c key=value` | Config override (TOML value) | `sandbox_workspace_write.network_access=true` for Tier 2 |
| `--ephemeral` | Do not persist session files | NOT used for tasks (breaks resume); used for throwaway probes |
| `--skip-git-repo-check` | Allow running outside a git repo | NOT used by default (projects must be git repos) |
| `--ignore-user-config` / `--ignore-rules` | Skip user config / .rules | Not used in v1; user config stays authoritative |
| `--dangerously-bypass-approvals-and-sandbox` | No sandbox at all | NEVER used by Handsfree (forbidden by policy engine) |

Subcommands:

- `codex exec resume <SESSION_ID> [PROMPT]` — resume a previous session by id
- `codex exec resume --last [PROMPT]` — resume most recent session

Notes:

- `codex exec` has **no interactive approval mechanism**. Sandbox mode is the only
  guardrail per turn. This is why Handsfree's risk tiers are implemented as
  *per-turn sandbox selection + resume*, not as mid-turn approval callbacks.
- Exit codes are not exhaustively documented upstream; the adapter must treat
  the JSONL stream (`turn.completed` / `turn.failed` / `error` items) as the
  source of truth for outcome, and the process exit code only as a secondary signal.
  (Pinning exact exit-code behavior is a validation step in the adapter issue.)

## JSONL event stream (probe evidence)

Live probe (2026-07-08, codex 0.141.0):

```
$ codex exec --skip-git-repo-check --ephemeral -s read-only --json "Reply with exactly the word: ping"
{"type":"thread.started","thread_id":"019f4b79-4a15-75e0-997f-1d38f4af162d"}
{"type":"item.completed","item":{"id":"item_0","type":"error","message":"`[features].codex_hooks` is deprecated. ..."}}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_1","type":"agent_message","text":"ping"}}
{"type":"turn.completed","usage":{"input_tokens":19789,"cached_input_tokens":4480,"output_tokens":17,"reasoning_output_tokens":10}}
```

Confirmed event envelope: one JSON object per line, `type` discriminator.

Event types (probe + official non-interactive mode docs):

| Event `type` | Payload | Notes |
|---|---|---|
| `thread.started` | `thread_id` (string, UUID-like) | Persist this id for `resume` |
| `turn.started` | — | |
| `item.started` / `item.updated` / `item.completed` | `item` object | Progress narration source |
| `turn.completed` | `usage` {input_tokens, cached_input_tokens, output_tokens, reasoning_output_tokens} | Terminal for a turn |
| `turn.failed` | error info | Terminal, maps to task `failed` |
| `error` | message | Stream-level error |

Item `type` values (docs; superset may grow): `agent_message`, `reasoning`,
`command_execution`, `file_change`, `mcp_tool_call`, `web_search`, `todo_list`,
`error`. The decoder MUST tolerate unknown event and item types (forward
compatibility requirement — codex releases frequently).

Observed quirks the adapter must handle:

1. A deprecation warning arrived as an `item.completed` with `item.type == "error"`
   *before* `turn.started`, without failing the run. Items of type `error` are
   therefore NOT necessarily fatal; only `turn.failed` / stream `error` are.
2. `"Reading additional input from stdin..."` was printed when stdin was an
   inherited pipe. The adapter must spawn with stdin explicitly closed
   (or `/dev/null`) and pass the prompt as argv.
3. Session persistence: with `--ephemeral` nothing is persisted; without it,
   sessions live under `~/.codex/sessions` and `thread_id` is resumable.

## Structured final output (`--output-schema`)

Official docs confirm: `--output-schema ./schema.json` enforces JSON Schema
conformance on the **final agent message**. Combined with `-o <file>`, the final
message (pure JSON) is written to a file and still printed to stdout (also
visible as the last `agent_message` item in the JSONL stream).

This is the mechanism behind the "agent self-summarization" decision:
Handsfree ships a response schema (`status`, `voice_summary`, `question`,
`detail`, `proposed_next_action`) and reads `voice_summary` for TTS.
Fallback when schema output fails to parse: use raw last message + rule-based
truncation (see DESIGN.md §Voice output contract).

## Sandbox behavior relevant to risk tiers

- `read-only` is the default sandbox for `codex exec`.
- `workspace-write` allows writes inside the workspace; **network is denied by
  default** and can be enabled via config override
  (`-c sandbox_workspace_write.network_access=true`).
- `danger-full-access` disables sandboxing for the turn.

This yields the 4-step escalation ladder used by the approval policy engine:

```
read-only  →  workspace-write  →  workspace-write + network  →  danger-full-access
```

Escalation is applied only to a *single* `codex exec resume` turn with a
narrow instruction, never persisted as a default.

## Auth & environment preconditions

- Codex CLI auth is managed by the user (`codex login`), outside Handsfree.
  The app performs a preflight (`codex --version`, auth check) and surfaces
  actionable errors; it never handles or stores API credentials itself.
- Version compatibility: adapter is developed against 0.141.0 behavior; the
  preflight records the detected version and the docs define a tested-version
  range policy (warn on untested major changes).

## Sources

- `codex exec --help` output, codex-cli 0.141.0 (captured 2026-07-08)
- Live probe JSONL (captured 2026-07-08, this machine)
- Official non-interactive mode docs: https://developers.openai.com/codex/noninteractive
  (redirects to https://learn.chatgpt.com/docs/non-interactive-mode)
