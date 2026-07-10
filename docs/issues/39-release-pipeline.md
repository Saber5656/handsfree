# Title

Release pipeline: Developer ID signing, notarization, GitHub Release workflow

## Summary

Implement the release path (R11): `sign.sh` / `notarize.sh` / `release.sh`
with correct nested inside-out signing, the tag-triggered `release.yml`
producing a draft GitHub Release with a stapled, checksummed zip, and the
maintainer runbook for the human-only key setup.

## Context

Signing + notarization are v1 scope (R11); keys/certificates are configured
**manually by the maintainer** — agents document, never create or store
them. CLT-only tooling is verified (`codesign`, `notarytool`, `stapler` —
research doc). The bundle may contain nested executables (e.g. `fake-codex`
from 34): signing discovers them generically so this issue does not depend
on 34.

## Scope

- `scripts/{sign,notarize,release}.sh`, Makefile wiring,
  `.github/workflows/release.yml`, `docs/RELEASING.md`. Not: auto-update,
  Homebrew cask, DMG.

## Detailed Requirements

1. `sign.sh` (input env `SIGN_IDENTITY`; two modes):
   - **Developer ID mode** (identity string): sign inside-out — first every
     nested executable discovered under `Contents/MacOS/` (all Mach-O files
     other than the main binary; hardened runtime, `--timestamp`, no
     entitlements), then the main app bundle with `--options runtime
     --timestamp --entitlements Resources/Handsfree.entitlements`. No
     `--deep`.
   - **Ad-hoc mode** (`SIGN_IDENTITY=-`): same order, but **omit
     `--timestamp`** (timestamping is unavailable for ad-hoc signatures)
     and omit hardened-runtime enforcement checks.
   - Verify: `codesign --verify --strict --deep` on the bundle AND
     `codesign --verify` on each nested binary; print `codesign -dvv`.
2. `notarize.sh`: `ditto -c -k --keepParent` zip; submit with
   `xcrun notarytool submit --wait` using EITHER an App Store Connect API
   key — decode `NOTARY_KEY_P8_BASE64` to a `mktemp` file chmod 600, call
   `--key <tmp> --key-id "$NOTARY_KEY_ID" --issuer "$NOTARY_KEY_ISSUER"`,
   and delete the temp key in a `trap … EXIT` cleanup — OR a keychain
   profile (`NOTARY_PROFILE`). On failure, fetch and print the notarization
   log. Then `xcrun stapler staple`, re-zip the stapled bundle as
   `Handsfree-<version>.zip`.
3. `release.sh`: flags `--skip-notarize` and `--allow-untagged` (local dry
   runs); default behavior enforces clean tree AND `VERSION` == tag
   `v<ver>` at `HEAD`; steps: `make app SIGN=none` (release config) →
   `sign.sh` → (`notarize.sh`) → `shasum -a 256` → `SHA256SUMS.txt` → print
   artifact paths.
4. `release.yml`: trigger tag `v*`; `permissions: contents: write` only;
   macos-26 runner; SHA-pinned actions; steps: full-depth checkout, import
   cert into a throwaway keychain from `MACOS_CERT_P12_BASE64` +
   `MACOS_CERT_PASSWORD` (exact `security` commands documented; keychain
   deleted in an `always()` cleanup step), run `release.sh`, upload artifact
   + `SHA256SUMS.txt` to a **draft** GitHub Release (manual publish gate —
   merge ≠ release).
5. `docs/RELEASING.md`: prerequisites (Apple Developer Program, Developer
   ID Application cert); one-time HUMAN-ONLY setup (cert export → base64,
   App Store Connect API key creation, `gh secret set` commands — explicitly
   marked as maintainer manual steps); cut-a-release steps (bump VERSION,
   tag, watch workflow, verify draft: download → `spctl -a -t exec -vv` →
   fresh-VM Gatekeeper check per 37's audit → publish); local fallback when
   CI notarization is flaky (known unknown #6): run `release.sh` locally
   with the same env vars.
6. CI amendment (02): PR builds additionally run
   `SIGN_IDENTITY=- ./scripts/sign.sh dist/Handsfree.app` after the app
   smoke — catches structural signing breakage early, no secrets on PRs.

## Acceptance Criteria

- [ ] `SIGN_IDENTITY=- ./scripts/release.sh --skip-notarize --allow-untagged`
      succeeds locally end-to-end (zip + checksums; structure verified).
- [ ] With the maintainer's identity on the dev machine: full
      sign→notarize→staple pass; `spctl -a -t exec -vv` returns "Notarized
      Developer ID" (evidence in PR, identifiers redacted).
- [ ] Nested-binary discovery: a dummy extra executable placed in
      `Contents/MacOS/` gets signed and verified; absence of any nested
      binary also passes.
- [ ] Test tag push → DRAFT release with artifact + SHA256SUMS (workflow run
      linked; draft deleted after).
- [ ] Keychain and temp-key cleanup run on failure paths (forced-fail run
      linked; `always()` verified).
- [ ] Version/tag mismatch and dirty tree are rejected without
      `--allow-untagged`.
- [ ] `RELEASING.md` reviewed by the maintainer (human sign-off comment).

## Validation

Script runs + workflow runs linked in the PR; release-time audit columns
hand off to 37's checklist.

## Dependencies

02, 05.

## Non-goals

Publishing (human gate), auto-update, Homebrew cask (v2), SLSA provenance
(v2 note in RELEASING.md), bundling fake-codex (34).

## Design References

DESIGN.md §10, §9.2 (T8/T9); R11; ADR-006, ADR-010; research doc
(notarytool); repository rule: secrets are configured manually by the human
maintainer.
