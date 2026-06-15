# Changelog

All notable changes to this project are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow
[SemVer](https://semver.org/). Major versions get a moving `vN` tag callers pin.

## [1.0.0] - 2026-06-15

Initial release. A reusable macOS Swift app release pipeline for the "SwiftKit"
pattern, shipped two ways.

### Added
- **Reusable workflow** `.github/workflows/release.yml` (`on: workflow_call`) -
  the primary consumption path. One macOS job: `actions/checkout`
  (fetch-depth 0) -> `actions/cache` (SwiftPM) -> `michielvha/gitversion-tag-action@v6`
  (compute SemVer + tag) -> the composite build/sign/package action -> optional
  Developer-ID keychain import + notarize/staple -> `gh release create
  v<semVer> --generate-notes <zip>`. Top-level `permissions: {}`, job
  `permissions: contents: write`. Inputs mirror the composite plus
  `package-path`, `runner`, `gitversion-config`, `tag-prefix`, `notarize`, with
  an all-optional signing/notary secret set.
- **Composite action** `action.yml` - the pure, secret-free build/sign/package
  step for custom pipelines. Clean Release `xcodebuild`
  (`-skipPackagePluginValidation`), build + embed SwiftPM helper executables
  into `Contents/Helpers/`, `codesign --force --deep` re-seal + verify,
  `ditto`-zip + sha256. `version` is an input; outputs `zip-path`, `version`,
  `sha256`. Identity-aware codesign flags (ad-hoc `--timestamp=none`; Developer
  ID `--timestamp --options runtime`).
- **Op-or-fail release gate**: `gitversion-tag-action` always runs
  `git tag <prefix><semVer> && git push`; a new version tags and the run
  proceeds, a no-bump push fails the tag step and the run stops with no Release.
  No synthetic always-run no-op or separate bump-gate output.
- `examples/release.yml` thin caller (`uses:
  vhco-pro/swift-release-action/.github/workflows/release.yml@v1`,
  `secrets: inherit`).
- Reference `gitversion.yml` (for the consumer's repo root, not used by this
  repo), `.github/dependabot.yml` (github-actions ecosystem). Third-party
  actions SHA-pinned; `gitversion-tag-action` stays `@v6`.
- Apache-2.0 license.

[1.0.0]: https://github.com/vhco-pro/swift-release-action/releases/tag/v1.0.0
