# Title

AudioEngineManager: microphone capture lifecycle with strict session binding

## Summary

Implement `HandsfreeSpeech/Audio/AudioEngineManager`: an actor owning one
`AVAudioEngine`, exposing an async buffer stream while a session is active and
guaranteeing zero audio objects held outside sessions.

## Context

DESIGN §4.1 and threat T10 require the mic to be verifiably off outside
sessions. Endpointing (09) and STT (08) consume this manager's buffers.
Prewarm budget: hotkey → listening ≤ 500 ms (DESIGN §15).

## Scope

- Capture lifecycle, device-change handling, buffer stream, state reporting.
- Not: STT/VAD consumption (08/09), TTS output (10), UI indicators.

## Detailed Requirements

1. Public API (actor `AudioEngineManager`):
   ```swift
   func start() async throws -> AsyncStream<CapturedBuffer>   // installs tap
   func stop() async                                          // removes tap, stops+resets engine
   var state: AudioCaptureState { get }   // .stopped | .starting | .capturing
   var events: AsyncStream<AudioEvent>    // .deviceChanged(name), .interrupted(reason), .resumed
   ```
   `CapturedBuffer` wraps `AVAudioPCMBuffer` + `AVAudioTime` + format info
   (single definition used by STT/VAD; no raw pointer exposure).
2. Tap: `inputNode.installTap(onBus: 0, bufferSize: 4096, format: nil)` using
   the node's native format; forward buffers into the stream via a bounded
   continuation (`bufferingPolicy: .bufferingNewest(8)`) — real-time thread
   must never block.
3. `stop()` guarantees: tap removed, engine stopped AND `reset()` called, the
   stream finished, and no `AVAudioEngine` reference retained beyond the actor
   (engine is recreated per `start()`; simplest correct way to release the
   HAL device). Add a debug assertion hook that `state == .stopped` implies
   `engine == nil`.
4. Device change: observe `AVAudioEngineConfigurationChange` notification;
   on change during capture: tear down, rebuild, restart tap, emit
   `.deviceChanged`; buffers from the old device are dropped (DESIGN §4.1 —
   current utterance is discarded by the consumer on this event).
5. Failure modes (DESIGN §12): input device unavailable, tap install throw,
   engine start throw → `start()` throws typed `AudioError` cases with
   user-presentable `errorDescription` keys (localized strings come later;
   use string keys now).
6. Prewarm: `start()` measures and logs (Log.audio) the elapsed time to first
   buffer; expose it on the stream's first element metadata for the perf test
   in issue 38.
7. No file writes of audio anywhere (grep-able acceptance criterion).

## Acceptance Criteria

- [ ] State transitions stopped→starting→capturing→stopped covered by unit
      tests using a protocol seam (`AudioEngineProviding`) so the real engine
      isn't needed; the real-engine path is covered by a `live`-tagged test.
- [ ] `live` test (manual, real mic): 3 s capture receives ≥ 20 buffers, then
      `stop()`; re-`start()` works (restartability).
- [ ] Device-change simulation via the notification (postable in tests)
      triggers rebuild + `.deviceChanged` event exactly once per change.
- [ ] After `stop()`, allocations hold no AVAudioEngine (weak-ref test).
- [ ] `grep -rn "write" Sources/HandsfreeSpeech/Audio` shows no audio
      persistence path.

## Validation

`swift test --filter AudioEngineManagerTests`; run the `live` test locally with
the assembled bundle (issue 05) and paste the log line showing time-to-first-
buffer ≤ 500 ms.

## Dependencies

01.

## Non-goals

Echo cancellation / voice processing (v2, ADR-008), output routing, sample-rate
conversion policy beyond native format pass-through.

## Design References

DESIGN.md §4.1, §9.2 (T10), §12, §15; ADR-008.
