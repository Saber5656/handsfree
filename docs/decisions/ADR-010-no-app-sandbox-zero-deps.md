# ADR-010: No App Sandbox; hardened runtime; zero third-party runtime dependencies

- Status: Accepted (2026-07-08)

## Context

Handsfree's core function is spawning a developer CLI (`codex`) against
arbitrary user-chosen project directories. The macOS App Sandbox would block or
severely complicate that (inherited sandbox for children, security-scoped
bookmarks for every project path), while providing little real containment for
a tool whose purpose is local code modification. Separately, as an OSS app
that listens to a microphone and touches source trees, supply-chain trust is a
first-order concern.

## Decision

1. **App Sandbox: off** in v1. Ship with **hardened runtime**, the single
   `com.apple.security.device.audio-input` entitlement, Developer ID signing,
   and notarization (R11). Mac App Store distribution is a non-goal.
2. **Zero third-party runtime dependencies**: production targets may import
   Apple frameworks only. Test-only dev dependencies require explicit review;
   GitHub Actions are pinned by commit SHA.

## Rationale

- Real containment of agent actions comes from codex's own sandbox tiers
  (ADR-004); App Sandbox would mostly fight the product, not the threats.
- Zero-dependency Swift is practical here (all needed capabilities are
  first-party) and makes the trust story auditable: what you build is Apple
  SDK + this repo.

## Consequences

- v1 cannot ship on the Mac App Store (accepted non-goal).
- Convenience libraries (hotkey helpers, settings frameworks, Sparkle) are
  excluded; equivalent functionality is implemented in-repo (hotkey via
  `RegisterEventHotKey`, no auto-update in v1).
- SECURITY.md documents the signing chain and how to verify artifacts.
