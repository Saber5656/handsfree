# Title

SpeechOutputArbiter: half-duplex gate, priority queue, earcons

## Summary

Implement the `SpeechOutputArbiter` actor that owns all sound output: it
serializes TTS utterances by priority, gates STT around playback
(half-duplex), drops stale narration, and plays the earcon set.

## Context

ADR-008 makes this component the guarantee that Handsfree never transcribes
its own voice — which the approval threat model depends on (T1/T3: the
approval earcon and template announcements flow through here).

## Scope

- `HandsfreeSpeech/Arbiter/SpeechOutputArbiter.swift`, `Earcons/` player +
  bundled placeholder sounds + generator script. Not: what to say (21/23),
  TTS engine (10).

## Detailed Requirements

1. API (actor):
   ```swift
   public init(tts: any TTSProvider, earconPlayer: any EarconPlaying,
               clock: any Clock<Duration>,
               staleAfter: Duration = .seconds(10),
               gateDebounce: Duration = .milliseconds(150))
   public func enqueue(_ u: SpokenUtterance)
   public func play(_ e: Earcon) async
   public func cancelAll(below priority: SpeechPriority)
   public var isOutputActive: Bool { get }
   public var sttGate: AsyncStream<Bool>
   public var events: AsyncStream<ArbiterEvent>
       // .spoke(SpokenUtterance) | .droppedStale(SpokenUtterance)
       // | .cancelledBeforeStart(SpokenUtterance) | .playedEarcon(Earcon)
   ```
   Internally each queued item is wrapped as `QueuedUtterance { utterance,
   enqueuedAt }` — `SpokenUtterance` itself is NOT modified (its shape is
   owned by issue 10).
2. `sttGate` contract: single-consumer stream; emits the current value
   (`true`) immediately on subscription; thereafter emits only on change;
   never finishes while the arbiter lives. `false` is emitted before any
   output starts; `true` after the queue drains AND `gateDebounce` elapses
   with no new output (so bursts don't flap the mic).
3. Queue rules:
   - `.approval` enqueue: current + queued `.narration` items are removed —
     the in-flight one via `tts.stop()` (its stream emits `.cancelled`),
     queued ones via `events .cancelledBeforeStart`; then the approval
     earcon + utterance play next.
   - `.narration` staleness: items whose `enqueuedAt` is older than
     `staleAfter` at dequeue are dropped with `.droppedStale`.
   - `.result` never drops; FIFO within a priority; higher priority first.
4. Earcons: `enum Earcon { sessionStart, sessionEnd, listeningResumed,
   dispatch, approvalRequest, success, failure, deviceChanged }` mapped to
   bundled `.caf` files (`Sources/HandsfreeSpeech/Resources/earcons/`).
   `approvalRequest` must be clearly distinct (longer, two-tone). Earcon
   playback holds the STT gate like any output.
5. Placeholder earcon generation: commit `scripts/gen-earcons.swift` — a
   single-file Swift script (run once with `swift scripts/gen-earcons.swift`,
   CLT-only) that synthesizes distinct sine patterns per earcon, writes
   16-bit PCM WAV via Foundation byte assembly, then shells to `afconvert`
   for `.caf`. The generated `.caf` files are committed; the script is kept
   for regeneration. No Xcode, no third-party tools.
6. Long-output courtesy: after any gated output ≥ 5 s, `listeningResumed` is
   appended as the FINAL gated output — `sttGate=true` is emitted only after
   it finishes plus the debounce (ordering is part of the half-duplex
   invariant; test it).

## Acceptance Criteria

- [ ] Priority/flush: approval removes in-flight narration (`tts.stop`
      observed on the mock) and queued narration (`.cancelledBeforeStart`);
      result items survive.
- [ ] Staleness: narration enqueued > 10 s ago (TestClock) is dropped with
      `.droppedStale`, never spoken.
- [ ] Gate: burst of 3 items → exactly one false→true cycle; initial `true`
      on subscription; debounce verified with TestClock.
- [ ] Resumed-earcon ordering: ≥ 5 s output → `.playedEarcon(.listeningResumed)`
      strictly before the gate reopens.
- [ ] All 8 earcon cases map to distinct existing `.caf` resources loadable
      via `Bundle.module` (unit test); regeneration script runs clean
      (`swift scripts/gen-earcons.swift` output in PR).
- [ ] `ArbiterLiveTests` (manual): session-start earcon + one sentence +
      listening-resumed earcon audible in order.

## Validation

`swift test --filter SpeechOutputArbiterTests` (mock TTS + TestClock);
manual `ArbiterLiveTests` note in PR.

## Dependencies

06, 07, 10.

## Non-goals

Barge-in, ducking other apps' audio, volume UI, narration content decisions
(23).

## Design References

DESIGN.md §4.4, §4.5, §5.4, §9.2 (T3); ADR-008.
