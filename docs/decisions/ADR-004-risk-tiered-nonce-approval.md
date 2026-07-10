# ADR-004: Risk-tiered approvals with random nonce echo

- Status: Accepted (2026-07-08, confirmed with product owner)

## Context

Voice is a spoofable, error-prone input channel: misrecognition, third-party
voices, replayed audio, and TTS-mediated prompt injection can all try to
trigger destructive actions. `codex exec` offers per-turn sandbox modes as the
enforcement primitive.

## Decision

Four risk tiers mapped to codex sandbox modes (DESIGN §6.5):
T0 read-only · T1 workspace-write (default for tasks) · T2 +network ·
T3 danger-full-access. New tasks always start at T1. Escalation is reactive
(agent reports `blocked`), single-turn, and approval-gated:

- T2: spoken echo of a per-approval random two-digit nonce ("承認 4 9").
- T3: nonce echo **plus** an on-screen button click (default; configurable off
  with an explicit risk warning).
- Approval announcements are built only from policy-layer templates plus the
  sanitized `blocked_action` field, always preceded by a distinct earcon.

## Rationale

- A random nonce defeats replayed recordings and accidental phrase matches;
  exact-match digits defeat fuzzy misrecognition (threats T1/T2).
- Template-only announcements + earcon prevent agent output from imitating an
  approval prompt (threat T3).
- The physical click at T3 is a deliberate hands-free break: full system
  access should require proof of presence at the machine.

## Consequences

- The common `push` follow-up costs one nonce echo — accepted UX tax.
- No pre-granting of elevated tiers at dispatch in v1 (revisit in v2).
- The approval engine is a security-critical component with its own exhaustive
  test issue and abuse-case acceptance criteria.
