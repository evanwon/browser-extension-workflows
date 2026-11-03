# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Start

**Project**: Shared CI/CD workflows and version tooling for browser extensions
**Status**: Production — consumed by [fancy-links](https://github.com/evanwon/fancy-links) and [gr-shelf-position-editor](https://github.com/evanwon/goodreads-shelf-position-editor)

This repo has no test suite or build step. Changes are released by tagging a version; consumers pin its commit SHA.

## Project Structure

```
browser-extension-workflows/
├── .github/workflows/
│   ├── build-release.yml       # Reusable: build, sign, AMO submit, GitHub release
│   └── test-pr.yml             # Reusable: test, lint, audit for PRs
├── release-tools/              # Lockfiles for web-ext and chrome-webstore-upload-cli, installed with npm ci
├── tools/
│   ├── version-bump.js         # CLI: extension-version-bump
│   └── validate-versions.js    # CLI: extension-validate-versions
├── package.json                # npm package with bin entries (no dependencies)
└── CLAUDE.md
```

The workflows are GitHub Actions YAML; the tools are standalone Node.js scripts with no dependencies.

## How Consumers Use This

Consumer extensions install this as a devDependency (`github:evanwon/browser-extension-workflows`) and set up:

1. **npm scripts** — `version:bump` → `extension-version-bump`, `version:check` → `extension-validate-versions`, `test:ci` → their test runner
2. **Thin workflow files** — `.github/workflows/test-pr.yml` and `build-release.yml` that `uses:` the reusable workflows from this repo, pinned to a release's commit SHA
3. **A `release` environment** — holds the store keys, and only `v*` tags may deploy to it
4. **`src/manifest.json`** — must have both `version` and `version_name` fields

## Version Encoding (Critical)

Firefox requires numeric-only `version` fields. Pre-releases use a 4-segment encoding:

- Stable `1.5.0` → `version: "1.5.0"`, `version_name: "1.5.0"`
- Pre-release `1.5.0-rc1` → `version: "1.4.5.1"`, `version_name: "1.5.0-rc1"`

The 4-segment version uses the **previous stable** as its base, not the target version. This ensures Firefox sorts `1.4.5.1 < 1.5.0`. The 4th segment equals the suffix number (`rc1` → `.1`, `rc2` → `.2`).

Both CLI tools enforce this encoding. `version-bump.js` computes it automatically; `validate-versions.js` verifies it.

## CLI Tool Architecture

### `tools/version-bump.js`

- Reads/writes `src/manifest.json` and `package.json` (in the consumer's working directory)
- Commands: `patch`, `minor`, `major`, `rc <type>`, `rc` (increment), `stable` (promote), or explicit version string
- Flags: `--dry-run`, `--no-git`
- Default behavior: writes files, `git add`, `git commit`, `git tag`
- Checks for existing tag before making changes
- Exports `parseVersion`, `getCurrentStableVersion`, `computePreReleaseVersion`, `bumpVersion` for testing

### `tools/validate-versions.js`

- Reads `src/manifest.json`, `package.json`, optionally `manifests/chrome/manifest.json`
- Validates: version consistency, version_name presence, pre-release 4-segment encoding, Chrome template match
- Chrome template mode: if `version === '__VERSION__'`, it's a placeholder — always valid
- Exports `validate`, `validateChromeTemplate` for testing

## Workflow Design Decisions

- **`npm ci --ignore-scripts`** — prevents malicious postinstall scripts in CI
- **Store keys never share a runner with third-party code** — build-release's `build` job runs npm ci, tests, lint and build with no secrets; its `release` job runs only the store tools and reads the keys from the caller's `release` environment. The comment above its `jobs:` says why each piece is there
- **Store tools come from `release-tools/` lockfiles** — both workflows check out this repo at `job.workflow_sha` and `npm ci` web-ext (and chrome-webstore-upload-cli when CWS upload is on) from there, so the tool versions move with the workflow commit a consumer pins. Dependabot updates them monthly, skipping versions under 7 days old
- **Actions are pinned by commit SHA**, with the tag in a comment; Dependabot keeps them current
- **`dry-run`** — the release job checks the store keys with read-only API calls (AMO's profile endpoint, Google's token exchange) and signs, submits and releases nothing. A manual run that asks for nothing at all stops after the build
- **Changelog generation** — always relative to the last stable tag, using `git log`. The `fetch-depth: 50` + `git fetch --tags` at checkout ensures tags are available
- **AMO credentials are optional** — if absent, unlisted signing is skipped (produces unsigned build); a listed submission or a dry run fails instead
- **Security audit** — build-release hard-fails on production vulns (`--omit=dev`) but only warns on dev-only vulns. test-pr has a configurable strict mode

## Chrome Support

Chrome support is implemented as opt-in via `chrome-enabled: true` in both workflows:

- `validate-versions.js` handles Chrome manifest templates at `manifests/chrome/manifest.json` with `__VERSION__` placeholders
- `build-release.yml` adds Chrome build, MV3 validation, ZIP packaging, Chrome Web Store submission, and dual-asset releases
- `test-pr.yml` adds Chrome build and MV3 validation steps
- All Chrome steps are gated by `if: inputs.chrome-enabled` — repos that don't opt in are completely unaffected
- Consumer extensions provide their own `build:chrome` npm script (e.g., `node tools/build.js --browser=chrome`)

## Releasing Changes

1. Make changes on `master`
2. Tag the merge commit with a version (e.g., `v1.2.0`) and move the floating `v1` tag to it:
   ```bash
   git tag v1.2.0 && git push origin v1.2.0
   git tag -f v1 && git push origin v1 --force
   ```
3. Consumers pin the commit SHA, not the tag, so nothing changes for them until they update the pin (Dependabot opens
   that PR in each consumer). A tag can be moved; a SHA can't, so a pushed tag can't change what a release runs
