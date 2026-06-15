# Plan - SwiftKit Build & Package Action

Spec: `docs/specs/swift-release-action.spec.md`
Status: complete

## Architecture decision

Two layers:

- **Reusable workflow** (`.github/workflows/release.yml`, `on: workflow_call`) -
  the primary consumption path. A single macOS job orchestrates the whole
  release: checkout → SwiftPM cache → compute version & tag (owner's
  `gitversion-tag-action@v6`) → the composite build/sign/package action →
  optional notarize+staple → `gh release create`. Consumers pin it with a thin
  ~10-line caller (`secrets: inherit`).
- **Composite action** (`action.yml`) - the pure, secret-free build +
  embed-helpers + sign + package step (mirroring ssm-connect's
  `scripts/release.sh`). `version` is an input; outputs `zip-path`, `version`,
  `sha256`. Unchanged; available for custom pipelines.

Versioning/tagging is the owner's existing `michielvha/gitversion-tag-action`
(not reinvented). The GitHub Release lives **inside** the reusable workflow.
`gitversion-tag-action` is the single SemVer source; its `git tag`/`git push` is
**op-or-fail** and IS the gate (new version → tags + continues; no-bump →
`git tag` fails → run stops). No synthetic always-run no-op.

## Phases

### Phase 1 - composite `action.yml`
- [x] Metadata + branding.
- [x] Inputs: `app-name`, `scheme`, `xcode-project`, `info-plist`,
  `helper-products`, `helper-package-path`, `entitlements`, `xcodegen`,
  `signing-identity`, `configuration`, `version`, `build-number`,
  `dist-dir`, `derived-data-path`.
- [x] Outputs: `zip-path`, `version`, `sha256`.
- [x] Steps (composite, `using: composite`):
  1. Install XcodeGen if `xcodegen == true` (`brew install xcodegen`, idempotent).
  2. Select newest installed Xcode.
  3. Stamp `Info.plist` when `version`/`build-number` provided.
  4. `xcodegen generate` when enabled.
  5. Clean Release build: `xcodebuild build -project -scheme -configuration`
     `-destination platform=macOS,arch=arm64 -derivedDataPath`
     `-skipPackagePluginValidation` + `CODE_SIGN_IDENTITY` /
     `CODE_SIGNING_REQUIRED=NO` / `CODE_SIGNING_ALLOWED=YES`.
  6. Build + embed each `helper-products` entry via
     `swift build -c release --product <p> --package-path <hpp>`, copy the
     binary into `<App>.app/Contents/Helpers/`, `chmod +x`, `codesign --force`.
  7. Re-sign the whole app `--force --deep` (with `--entitlements` if given),
     then `codesign --verify --deep --strict`. Identity-aware codesign flags:
     ad-hoc `-` → `--timestamp=none`; Developer ID → `--timestamp --options
     runtime` (notarization-ready).
  8. `ditto -c -k --sequesterRsrc --keepParent` → `dist/<app>-<ver>.zip` +
     `.sha256`; write outputs to `$GITHUB_OUTPUT`.
- Bash, `set -euo pipefail`, all paths from inputs; resolve version from plist
  when input empty (parity with `release.sh`).

### Phase 2 - reusable workflow + thin caller
- [x] `.github/workflows/release.yml` (`on: workflow_call`): single macOS job,
  `permissions: {}` top-level + job `contents: write`, steps:
  1. `actions/checkout` (fetch-depth 0, SHA-pinned).
  2. `actions/cache` for SwiftPM (`.build` + `~/Library/Caches/org.swift.swiftpm`,
     keyed off `Package.resolved`), SHA-pinned.
  3. `michielvha/gitversion-tag-action@v6` (id `version`) with `configFilePath` +
     `prefix` → `semVer`. Op-or-fail tag gate (no synthetic no-op).
  4. Optional Developer-ID keychain import - `if: inputs.notarize`.
  5. `uses: vhco-pro/swift-release-action@v1` with
     `version: ${{ steps.version.outputs.semVer }}` + the build inputs.
  6. Optional notarize + staple of the zip - `if: inputs.notarize`.
  7. `gh release create v<semVer> --generate-notes <zip>` with
     `GH_TOKEN: ${{ github.token }}`.
  - Inputs mirror the composite (`app-name` required, scheme, xcode-project,
    info-plist, helper-products, helper-package-path, entitlements, xcodegen,
    signing-identity) plus `package-path`, `runner`, `gitversion-config`,
    `tag-prefix`, `notarize`. Secrets: the signing/notary set, all optional.
- [x] `examples/release.yml`: thin caller -
  `jobs.release.uses: vhco-pro/swift-release-action/.github/workflows/release.yml@v1`
  with `with:` inputs, `secrets: inherit`, job `permissions: contents: write`.
  Shows how `claude-companion` consumes it.

### Phase 3 - docs & meta
- [x] `README.md`: reframed - primary consumption is the reusable workflow
  (gitversion via the owner's action + build + GitHub Release); op-or-fail gate
  stated; the composite is available for custom pipelines; reusable-workflow
  inputs/secrets tables + composite inputs/outputs tables; thin-caller snippet;
  signing tiers.
- [x] `LICENSE` (Apache-2.0, ssm-connect header style → `VH & Co BV`).
- [x] `CHANGELOG.md` (Keep a Changelog).
- [x] `.github/dependabot.yml` (github-actions ecosystem, weekly, `ci` prefix).
- [x] `gitversion.yml` kept as **reference only** for the consumer's repo root
  (clearly labelled "not used here"); read by `gitversion-tag-action`.

## Risks / unverifiable
- No macOS runner in this env → the composite + reusable workflow are authored
  to the proven ssm-connect commands but not end-to-end executed here.
- Developer ID / notarization path (keychain import + `notarytool` + `stapler`)
  cannot be exercised without a cert + Apple account.
- `actionlint`/`yamllint` not run (sandbox); reusable-workflow `inputs`/`secrets`
  schema and the op-or-fail gate behavior reviewed by hand; end-to-end validation
  happens on first real consumer run.
