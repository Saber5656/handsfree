# ADR-008: Half-duplex audio in v1 (no barge-in)

- Status: Accepted (2026-07-08)

## Context

While the app speaks (TTS), the microphone would pick up the app's own voice.
True barge-in (interrupting TTS by speaking) requires acoustic echo
cancellation (`setVoiceProcessingEnabled` voice-processing I/O) whose behavior
varies across output devices (speakers vs AirPods vs external DACs) and would
sit on the critical path of every session.

## Decision

v1 is **half-duplex**: the `SpeechOutputArbiter` pauses the STT stream during
any TTS playback and resumes it afterwards (with a subtle listening-resumed
earcon on long narrations). Users interrupt long playback with the hotkey
(which also serves as panic-stop); voice barge-in is a v2 item behind the same
arbiter interface.

## Rationale

- Deterministic: zero risk of the app approving/canceling based on its own
  speech — this interacts directly with the approval threat model (T1/T3).
- AEC quality is device-dependent and unverifiable in CI; shipping v1 on it
  would gate the golden path on the flakiest component.

## Consequences

- The user cannot talk over narration; narration is therefore aggressively
  throttled and stale lines are dropped (DESIGN §5.3, §4.4).
- Approval prompts are short by design so the await-echo window opens quickly.
- v2 barge-in slots in at the arbiter without FSM changes.
