# Spec - SwiftKit Build & Package Action (reusable composite GitHub Action)

Status: implemented
Slug: `swift-release-action`

## Problem

`vhco-pro/ssm-connect` established a repeatable pattern for shipping macOS Swift
menu-bar apps: a local Swift package is the source of truth for dependencies; a
thin XcodeGen-generated Xcode app target produces the signed `.app`; build
phases embed helper binaries into `Contents/Helpers/` and re-sign the bundle
ad-hoc; a `release.sh` does a clean Release build and `ditto`-zips it with a
sha256.

That **build/sign/package** logic was copy-pasted per repo. The next consumer is
`claude-companion` (another macOS menu-bar Swift app). We want the mechanical
build step extracted into ONE reusable artifact consumers pin with `@v1`, so it
is maintained once.

## Design

This artifact ships **two layers**, and the orchestration belongs in the repo
(not in every consumer):

- **Reusable workflow** (`.github/workflows/release.yml`, `on: workflow_call`) -
  the **primary** deliverable. One macOS job composing checkout → compute version
  & tag (owner's `gitversion-tag-action`) → build/sign/package (the composite) →
  GitHub Release (`gh release create`). Consumers pin it with a ~10-line caller.
- **Composite action** (`action.yml`) - the pure, secret-free build → embed
  helpers → sign → `ditto`-zip + sha256 step, with **`version` as an input** and
  outputs `{zip-path, sha256, version}`. **No GitVersion logic, no
  release-publishing.** Available for custom pipelines (e.g. CI smoke builds).
- **Versioning + tagging** uses the owner's existing
  [`michielvha/gitversion-tag-action`](https://github.com/michielvha/gitversion-tag-action)
  (computes SemVer from conventional commits AND tags; output `semVer`).
- **GitHub-Release creation** is `gh release create` inside the reusable workflow.
- `examples/release.yml` is the canonical thin caller.

## Op-or-fail release gate

`gitversion-tag-action` is a composite that runs gittools gitversion
setup/execute, outputs `semVer`, and **always** runs
`git tag <prefix><semVer> && git push origin <tag>`. There is no "should-release"
output. Its success/failure **is** the gate:

- A real bump → new tag → created + pushed → the reusable workflow continues to
  build and publish the Release.
- A no-bump push → SemVer resolves to the already-tagged version → `git tag`
  errors on the existing tag → the step **fails** → the run stops (no build, no
  Release). Intended. We do **not** add a synthetic always-run no-op to swallow
  that failure.

## Goal

A standalone public repo providing the reusable release **workflow** as the
primary consumption path, plus the underlying **composite action** for the pure
build + embed + sign + package step (reusable standalone, e.g. CI smoke builds).

## Reference (do-not-modify)

- `vhco-pro/ssm-connect` PR branch `refactor/spm-feature-package` and local
  clone at `/Users/michielvh/code/one-b2c/ssm-connect`.
- Key learnings extracted from it:
  - Local Swift package is the dependency source of truth; XcodeGen renders a
    thin app target around it.
  - `xcodebuild ... -skipPackagePluginValidation` is REQUIRED (SwiftPM
    build-tool plugins like smithy-swift's otherwise fail headless).
  - Runner's default Xcode can be too old for XcodeGen's emitted project format
    (objectVersion 77) - must select the newest installed Xcode.
  - Embedding a helper binary AFTER Xcode signs invalidates the outer seal, so
    re-sign helper(s) then the whole app with `codesign --force --deep`, then
    `codesign --verify --deep --strict`.
  - Ad-hoc identity `-` (no Apple team) for the personal/v1 tier; NOT notarized.
  - SwiftPM + build products cached; `ditto -c -k --sequesterRsrc --keepParent`
    for the zip.
  - Versioning/tagging via the owner's `michielvha/gitversion-tag-action`,
    not re-implemented here.
  - All third-party actions pinned to commit SHAs.

## Functional requirements

- **FR-0 Reusable workflow** at `.github/workflows/release.yml`
  (`on: workflow_call`). Single macOS job; top-level `permissions: {}`, job
  `permissions: contents: write`. Steps: checkout (`fetch-depth: 0`) → SwiftPM
  cache → `gitversion-tag-action@v6` (op-or-fail tag gate; id `version`,
  output `semVer`) → optional Developer-ID keychain import (when `notarize`) →
  the composite `action.yml` with `version: ${{ steps.version.outputs.semVer }}`
  → optional notarize + staple → `gh release create v<semVer> --generate-notes
  <zip>` with `GH_TOKEN: ${{ github.token }}`. SHA-pin checkout + cache;
  `gitversion-tag-action` stays `@v6`. Inputs mirror the composite (plus
  `runner`, `gitversion-config`, `tag-prefix`, `package-path`, `notarize`);
  secrets are the optional signing/notary set.
- **FR-1 Composite action** at `action.yml` (`using: composite`), parameterized
  by inputs (below). Secret-free; does no versioning and no release-publishing.
- **FR-2 Build env**: install XcodeGen (when `xcodegen: true`) and select the
  newest installed Xcode.
- **FR-3 Version stamp**: when `version`/`build-number` are supplied, write them
  into the app's `Info.plist` (`CFBundleShortVersionString` /
  `CFBundleVersion`). `version` is an INPUT - the action does not compute it.
- **FR-4 Build + package**: generate the Xcode project (when `xcodegen: true`),
  clean Release build with `-skipPackagePluginValidation` and the configured
  signing identity, build any `helper-products` (SwiftPM executables) with
  `swift build -c release` and embed them into `<App>.app/Contents/Helpers/`,
  re-sign helpers + the app `--force --deep`, `codesign --verify --deep
  --strict`, then `ditto`-zip into `dist/<app-name>-<version>.zip` + a `.sha256`.
  Emit `zip-path`, `version`, `sha256` as outputs.
- **FR-5 Signing tiers**: ad-hoc `-` default (`--timestamp=none`); a real
  Developer ID identity → `--timestamp --options runtime` (notarization-ready).
  Cert import + notarize/staple are caller concerns (they need secrets).
- **FR-6 Thin caller example**: `examples/release.yml` showing how
  `claude-companion` consumes the reusable workflow -
  `jobs.release.uses: vhco-pro/swift-release-action/.github/workflows/release.yml@v1`
  with `with:` inputs, `secrets: inherit`, and job `permissions: contents:
  write`.

## Inputs (composite action)

| Input | Type | Default | Purpose |
|---|---|---|---|
| `app-name` | string | (required) | Product / `.app` base name; drives zip name |
| `scheme` | string | = `app-name` | Xcode scheme to build |
| `xcode-project` | string | `<app-name>.xcodeproj` | `-project` path |
| `info-plist` | string | `<app-name>/Info.plist` | plist to stamp |
| `helper-products` | string | `''` | space/newline list of SwiftPM executable products to build + embed |
| `helper-package-path` | string | `.` | `--package-path` for `swift build` of helpers |
| `entitlements` | string | `''` | entitlements file for the `--deep` re-sign |
| `xcodegen` | string | `true` | run `xcodegen generate` before building |
| `signing-identity` | string | `-` | codesign identity (ad-hoc `-` default) |
| `configuration` | string | `Release` | `xcodebuild` configuration |
| `version` | string | `''` | version to stamp (caller's computed SemVer); empty → keep plist value |
| `build-number` | string | `''` | `CFBundleVersion` value (typically `github.run_number`) |
| `dist-dir` | string | `dist` | output dir for the zip + sha256 |
| `derived-data-path` | string | `.build/DerivedData` | `xcodebuild -derivedDataPath` |

## Outputs

| Output | Purpose |
|---|---|
| `zip-path` | path to the packaged `.zip` |
| `version` | version that was built/packaged |
| `sha256` | SHA-256 of the `.zip` |

## Non-functional / constraints

- **NF-1** Third-party actions pinned to commit SHAs in the reusable workflow
  (`checkout`, `cache`); the owner's `gitversion-tag-action` pinned `@v6`.
- **NF-2** The composite action is secret-free; the reusable workflow's job (and
  the caller's job) grant `contents: write` (for tag push + Release).
- **NF-3** Default tier is ad-hoc, un-notarized - works with zero secrets and no
  Apple Developer account, identical locally and on CI.
- **NF-4** Notarization-ready codesign flags for a real Developer ID identity
  (caller does the actual notarize/staple).
- **NF-5** Apache-2.0 license, matching ssm-connect's header style.

## Out of scope

- Re-implementing GitVersion/gittools logic (delegated to the owner's
  `michielvha/gitversion-tag-action`).
- A synthetic bump-gate / always-run no-op (the tag step's op-or-fail IS the gate).
- Homebrew cask bump (consumer's `homebrew-tap` sync handles that downstream).
- Building/testing the Swift app itself (consumer-owned).

## Acceptance

- Reusable workflow authored at `.github/workflows/release.yml`
  (`on: workflow_call`): single macOS job, `permissions: {}` top-level + job
  `contents: write`, composing checkout → cache → `gitversion-tag-action@v6` →
  composite action → optional notarize → `gh release create`. Inputs/secrets as
  in FR-0; third-party actions SHA-pinned.
- Composite action unchanged; secret-free; outputs `zip-path`, `version`,
  `sha256`; `version` is an input.
- README reframed: primary consumption = the reusable workflow; the composite is
  available for custom pipelines; op-or-fail gate stated; thin-caller snippet
  shown.
- `examples/release.yml` is the canonical thin caller
  (`jobs.release.uses: .../release.yml@v1`, `secrets: inherit`).
