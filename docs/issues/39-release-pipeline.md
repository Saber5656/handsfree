# Title

Release pipeline: Developer ID signing, notarization, GitHub Release workflow

## Summary

Implement the full release path (R11): `make sign/notarize/release` scripts
with proper nested signing, the tag-triggered `release.yml` workflow producing
a draft GitHub Release with a stapled, checksummed zip, and the maintainer
runbook for the manual key setup.

## Context

Signing + notarization are v1 scope by owner decision (R11); keys/certificates
are configured **manually by the maintainer** (never generated or stored by
agents — repository rule). CLT-only tooling is verified available
(`codesign`, `notarytool`, `stapler` — research doc). The bundle contains a
nested executable (`fake-codex`, from 34) that must be signed correctly.

## Scope

- `scripts/sign.sh`, `scripts/notarize.sh`, `scripts/release.sh`, Makefile
  wiring, `.github/workflows/release.yml`, `docs/RELEASING.md` runbook.
- Not: auto-update, Homebrew cask, DMG styling (zip only in v1).

## Detailed Requirements

1. `sign.sh`:
   - Inputs (env): `SIGN_IDENTITY` ("Developer ID Application: …").
   - Order: sign nested `Contents/MacOS/fake-codex` first, then the main app
     bundle — both with `--options runtime --timestamp` and the entitlements
     file for the MAIN app only (fake-codex gets hardened runtime, no
     entitlements). **No `--deep`** (explicit inside-out signing).
   - Verify: `codesign --verify --strict --deep` + `codesign -dvv` output.
2. `notarize.sh`:
   - Zip via `ditto -c -k --keepParent`; submit
     `xcrun notarytool submit --wait` using EITHER App Store Connect API key
     env (`NOTARY_KEY_ID`, `NOTARY_KEY_ISSUER`, `NOTARY_KEY_P8_BASE64`) or a
     stored keychain profile (`NOTARY_PROFILE`) — support both, prefer API
     key in CI; on failure fetch and print the notarization log.
   - `xcrun stapler staple` the .app, re-zip the stapled bundle as the final
     artifact `Handsfree-<version>.zip`.
3. `release.sh`: version consistency check (`VERSION` == git tag `v<ver>`),
   clean-tree check, `make app` (release config), sign, notarize, produce
   `SHA256SUMS.txt`, output artifact paths.
4. `release.yml`:
   - Trigger: push of tag `v*`. `permissions: contents: write` only.
   - macos-26 runner; SHA-pinned actions; steps: checkout (full-depth for
     version check), import signing cert into a throwaway keychain from
     secrets `MACOS_CERT_P12_BASE64` + `MACOS_CERT_PASSWORD` (document the
     exact `security` commands; keychain deleted in an `always()` cleanup
     step), run `release.sh`, upload artifact + checksums to a **draft**
     GitHub Release (manual publish = the release gate; merge ≠ release).
   - Secrets are referenced by name only; `docs/RELEASING.md` documents how
     the MAINTAINER creates/registers them by hand (cert export, API key
     creation, `gh secret set` commands) — the runbook explicitly marks these
     as human-only steps.
5. `docs/RELEASING.md` runbook: prerequisites (Apple Developer Program,
   Developer ID cert), one-time setup, cut-a-release steps (bump VERSION,
   tag, watch workflow, verify draft, `spctl -a -t exec -vv` on a downloaded
   artifact, fresh-machine/VM Gatekeeper check per audit doc 37, publish),
   local fallback path when CI notarization is flaky (known unknown #6):
   run `release.sh` locally with the same env.
6. CI amendment (02): PR builds also run `sign.sh` in **ad-hoc mode**
   (`SIGN_IDENTITY="-"`) to catch structural signing breakage early (no
   secrets on PRs).

## Acceptance Criteria

- [ ] Local ad-hoc end-to-end: `SIGN_IDENTITY=- ./scripts/release.sh --skip-notarize`
      produces a verifiable zip (structure + checksums).
- [ ] With maintainer-provided identity on the dev machine: full
      sign→notarize→staple pass; `spctl -a -t exec -vv dist/Handsfree.app`
      returns "Notarized Developer ID" (evidence in PR, ids redacted).
- [ ] Tag push on a test tag produces a DRAFT release with artifact +
      SHA256SUMS (workflow run linked; draft deleted after).
- [ ] Keychain cleanup step runs on failure paths (`always()` verified by a
      forced-fail run).
- [ ] Nested fake-codex signature valid (`codesign --verify` on the nested
      binary specifically).
- [ ] `RELEASING.md` reviewed by the maintainer (human) for the manual key
      steps.

## Validation

Script runs + workflow runs linked in the PR; audit items handed to 37's
checklist for the release-time pass.

## Dependencies

02, 05 (and 34's bundled fake-codex).

## Non-goals

Publishing (human gate), auto-update, Homebrew cask (v2), SLSA provenance
(v2 candidate — note in RELEASING.md).

## Design References

DESIGN.md §10, §9.2 (T8/T9); R11; ADR-006, ADR-010; research doc (notarytool
availability); repository rule: secrets are configured manually by the human
maintainer.
