# Title

AudioEngineManager: microphone capture lifecycle with strict session binding

## Summary

Implement `HandsfreeSpeech/Audio/AudioEngineManager`: an actor owning one
`AVAudioEngine`, exposing an async buffer stream while a session is active
and guaranteeing zero audio objects held outside sessions.

## Context

DESIGN §4.1 and threat T10 require the mic to be verifiably off outside
sessions. Endpointing (09) and STT (08) consume this manager's buffers.
Prewarm budget: hotkey → listening ≤ 500 ms (DESIGN §15).

## Scope

- Capture lifecycle, device-change handling, buffer stream, state reporting,
  `CapturedBuffer` type definition. Not: STT/VAD consumption (08/09), TTS
  output (10), UI indicators, logging facade wiring (uses a local
  `os.Logger` until issue 04 lands, then a mechanical follow-up swap noted in
  code).

## Detailed Requirements

1. `CapturedBuffer` (defined here, used by 07/08/09):
   ```swift
   public struct CapturedBuffer: @unchecked Sendable {
       public let pcm: AVAudioPCMBuffer   // never mutated after creation
       public let time: AVAudioTime
       public let format: AVAudioFormat
   }
   ```
   `@unchecked Sendable` is justified in a doc comment: buffers are created
   on the tap thread, handed over exactly once, and treated as immutable by
   contract; consumers must not mutate `pcm`. (Copy-out was rejected for
   allocation cost on the audio path — note in comment.)
2. Public API (actor `AudioEngineManager`):
   ```swift
   func start() async throws -> AsyncStream<CapturedBuffer>
   func stop() async
   var state: AudioCaptureState { get }   // .stopped | .starting | .capturing | .stopping
   var events: AsyncStream<AudioEvent>    // .deviceChanged(name: String?), .interrupted(reason: String), .resumed
   ```
   Reentrancy table (each row tested): `start()` in `.capturing`/`.starting`
   → throws `AudioError.alreadyCapturing`; `start()` in `.stopping` → throws
   `AudioError.teardownInProgress` (caller debounces per DESIGN §12);
   `stop()` in any state → idempotent, always drains to `.stopped`.
3. Tap: `inputNode.installTap(onBus: 0, bufferSize: 4096, format: nil)`
   (native format); buffers forwarded via bounded continuation
   (`.bufferingNewest(8)`) — the render thread never blocks. Note: tap
   installation does not throw in AVFoundation; preflight instead — missing
   input device / zero-channel native format → `start()` throws
   `AudioError.noInputDevice` / `.formatUnavailable`; `engine.start()` errors
   propagate as `AudioError.engineStartFailed(underlying)`.
4. `stop()` guarantees: tap removed, engine stopped and `reset()`, stream
   finished, and the engine reference released (engine is recreated per
   `start()`); debug assertion that `.stopped` implies `engine == nil`.
5. Device change: observe `.AVAudioEngineConfigurationChange`; during
   capture: tear down, rebuild, restart tap; emit `.deviceChanged(name)`
   then `.resumed` on success, or `.interrupted(reason)` if the rebuild
   fails (then transition to `.stopped`). Buffers from the old device are
   dropped (consumer discards the current utterance on `.deviceChanged`).
6. Testability seam (define in this issue):
   ```swift
   protocol AudioEngineProviding {           // production: AVAudioEngine-backed
       func makeEngine() -> AudioEngineHandle
   }
   protocol AudioEngineHandle {              // start/stop/reset/installTap/removeTap,
       …                                      // nativeFormat, notification hooks
   }
   ```
   plus a `FakeEngineHandle` in `HandsfreeTestSupport` scripting formats,
   start failures, buffer emission, and configuration-change notifications.
   State-machine/unit tests use the fake; one `AudioEngineLiveTests` suite
   covers the real engine (live convention, issue 02).
7. Prewarm: `start()` logs elapsed time to first buffer; the first
   `CapturedBuffer` is preceded by a recorded timestamp accessible to the
   perf checks (expose `lastStartLatency: Duration?`).
8. No audio persistence: no `AVAudioFile`, `ExtAudioFile`, `FileHandle`,
   `OutputStream`, `FileManager`, or `.write(` in `Sources/HandsfreeSpeech`
   (checked in Validation; also enforced later by 37's static checks).

## Acceptance Criteria

- [ ] Reentrancy table fully tested via the fake engine (5+ cases).
- [ ] Device-change: notification → rebuild → `.deviceChanged` + `.resumed`
      exactly once per change (fake); rebuild failure → `.interrupted` +
      `.stopped`.
- [ ] After `stop()`, weak reference to the engine handle is nil.
- [ ] `AudioEngineLiveTests` (manual, real mic, run from the issue-05
      bundle): 3 s capture ≥ 20 buffers; stop; re-start works; log shows
      time-to-first-buffer ≤ 500 ms.
- [ ] Persistence grep (Req 8) returns nothing.

## Validation

`swift test --filter AudioEngineManagerTests` (CI-safe, fake-based);
`swift test --filter 'AudioEngineLiveTests'` run locally — output + latency
line pasted in PR.

## Dependencies

01.

## Non-goals

Echo cancellation/voice processing (v2, ADR-008), output routing,
sample-rate conversion (owned by the STT provider), `Log.audio` facade wiring
(mechanical swap after 04).

## Design References

DESIGN.md §4.1, §9.2 (T10), §12, §15; ADR-008; research doc (TCC/bundle note).
