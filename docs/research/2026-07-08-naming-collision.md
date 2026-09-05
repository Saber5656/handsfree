# Research: "handsfree" naming collision

- Date: 2026-07-08
- Status: informational; feeds the pre-release branding gate (see ISSUE_PLAN, wave R)

## Findings

| Namespace | Collision | Details |
|---|---|---|
| npm `handsfree` | **Taken** | Oz Ramos' Handsfree.js — browser face/hand/pose tracking library (v8.5.1). Project is archived/abandoned; package restored by third parties. |
| GitHub repos | **Taken (different domain)** | `CreativeInquiry/handsfree`, `MIDIBlocks/handsfree`, `BrowseHandsfree` org — all the same hand-tracking lineage. |
| Homebrew | **Free** | `brew search handsfree` → no formula; `brew info --cask handsfree` → "No Cask with this name exists" (checked 2026-07-08). |
| Domain association | Weak | handsfreejs.netlify.app exists; "handsfree" as a generic word is heavily used (car kits, headsets). |

## Assessment

- The colliding projects are **web hand-tracking libraries**, a different domain
  from "voice-driven coding agents on macOS". Confusion risk is moderate, not
  fatal; the archived status of Handsfree.js reduces active conflict.
- The generic word makes SEO/discoverability weak regardless of the collision.
- Homebrew cask name `handsfree` is available today, which matters for
  distribution UX (`brew install --cask handsfree`).

## Decision for v1 development

Keep **Handsfree** as the working title. Repository, bundle identifier prefix,
and user-facing strings are parameterized so a rename before public launch is a
bounded change (single constants file + docs sweep). A dedicated pre-release
issue (branding gate) re-evaluates:

1. Final name decision (keep vs. rename; candidate qualifiers if renamed).
2. Bundle identifier freeze.
3. Homebrew cask name availability re-check at that date.
4. Trademark/GitHub org squatting sanity check.

## Sources

- https://github.com/CreativeInquiry/handsfree
- https://github.com/MIDIBlocks/handsfree
- https://www.npmjs.com/package/handsfree
- Local `brew search` / `brew info --cask` output, 2026-07-08
