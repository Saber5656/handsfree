# ADR-005: Voice summaries via agent self-summarization (`--output-schema`)

- Status: Accepted (2026-07-08, confirmed with product owner)

## Context

Raw agent output is far too long to speak. Candidates for the TTS shortening
layer: (a) make the agent itself emit a structured voice summary, (b) call a
separate LLM API to summarize, (c) rule-based extraction only.

## Decision

(a): every turn passes `--output-schema` with the Handsfree response schema
(DESIGN Appendix C.1). The agent's final message must be JSON containing
`status`, `voice_summary` (1–3 sentences, user's language), and the
`question`/`blocked_*` fields that drive the dialogue FSM. A rule-based
fallback chain (DESIGN §6.3) handles non-conforming output.

## Rationale

- Zero additional dependencies, API keys, cost, or privacy surface — consistent
  with ADR-003; the summarizer is the same agent the user already trusts.
- `--output-schema` is a first-class codex feature (verified 0.141.0), giving
  schema-enforced output rather than prompt-hoping.
- The structured `status`/`blocked_*` fields double as the machine-readable
  needs-input/escalation signal — one contract serves narration AND control flow.

## Consequences

- Summary quality depends on the agent model; the scaffold (DESIGN §6.4)
  carries explicit style constraints, and `voice_summary` is sanitized and
  capped before speech (threat T3).
- If future codex versions change schema enforcement, the fallback chain and
  contract tests catch it (known unknown #4).
