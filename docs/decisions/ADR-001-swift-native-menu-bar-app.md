# ADR-001: Swift-native SwiftPM menu bar app

- Status: Accepted (2026-07-08, confirmed with product owner)
- Deciders: product owner + design agent

## Context

Handsfree is a resident macOS app whose core loop is low-latency audio capture,
on-device STT/TTS, and a global hotkey. Candidates: Swift native, Tauri
(Rust + web UI), Electron (TypeScript).

## Decision

Build natively in Swift: AppKit/SwiftUI shell, `AVAudioEngine` capture,
`Speech.framework` (SpeechAnalyzer/SpeechTranscriber) STT, `AVSpeechSynthesizer`
TTS, distributed as a menu bar app (`LSUIElement`).

## Rationale

- The differentiating requirements (on-device ja/en STT, VAD, half-duplex audio
  arbitration, mic-lifecycle guarantees) are first-party Apple APIs; FFI
  bridges (Tauri) or Node bindings (Electron) add cost and failure modes with
  no benefit.
- Resident-footprint budget (<80 MB RSS idle) rules out Electron.
- macOS-only is an accepted v1 constraint (R1); cross-platform reach was the
  main argument for Tauri and is not a v1 goal.

## Consequences

- Contributor pool is smaller than TS; mitigated by zero-dependency SwiftPM
  and heavily documented issues.
- Cross-platform port would be a rewrite of the shell layers; Core logic is
  kept UI-framework-free to preserve portability of the design.
- Toolchain constraint follows: see ADR-006 (SwiftPM + CLT-only pipeline).
