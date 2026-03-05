# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Start

**Project**: Shared CI/CD workflows and version tooling for browser extensions
**Status**: Production — consumed by [fancy-links](https://github.com/evanwon/fancy-links) and [gr-shelf-position-editor](https://github.com/evanwon/goodreads-shelf-position-editor)

This repo has no test suite or build step. Changes are released by moving the floating `v1` tag forward.

## Project Structure

```
extension-workflows/
├── .github/workflows/
│   ├── build-release.yml       # Reusable: build, sign, AMO submit, GitHub release
│   └── test-pr.yml             # Reusable: test, lint, audit for PRs
├── tools/
│   ├── version-bump.js         # CLI: extension-version-bump
│   └── validate-versions.js    # CLI: extension-validate-versions
├── package.json                # npm package with bin entries (no dependencies)
└── CLAUDE.md
```

There are only 4 functional files. The workflows are GitHub Actions YAML; the tools are standalone Node.js scripts with no dependencies.

## How Consumers Use This

Consumer extensions install this as a devDependency (`github:evanwon/extension-workflows`) and set up:

1. **npm scripts** — `version:bump` → `extension-version-bump`, `version:check` → `extension-validate-versions`, `test:ci` → their test runner
2. **Thin workflow files** — `.github/workflows/test-pr.yml` and `build-release.yml` that `uses:` the reusable workflows from this repo at `@v1`
3. **`src/manifest.json`** — must have both `version` and `version_name` fields

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
- **`web-ext` installed globally** in build-release (faster for repeated `build`/`sign` calls) but via `npx` in test-pr
- **Changelog generation** — always relative to the last stable tag, using `git log`. The `fetch-depth: 50` + `git fetch --tags` at checkout ensures tags are available
- **AMO credentials are optional** — if absent, signing is gracefully skipped (produces unsigned build). This lets the workflow run without secrets configured
- **Security audit** — build-release hard-fails on production vulns (`--omit=dev`) but only warns on dev-only vulns. test-pr has a configurable strict mode
- **Browser matrix** — `browser: [firefox]` is set up for future Chrome expansion

## Chrome Support (Coming Soon)

Chrome support is an active priority. Groundwork is already in place:

- `validate-versions.js` already handles Chrome manifest templates at `manifests/chrome/manifest.json` with `__VERSION__` placeholders
- The build workflow matrix has `browser: [firefox]` — designed to expand to `[firefox, chrome]`
- Consumer extensions should avoid Firefox-only APIs where possible
- Remaining work: add Chrome build/package steps to `build-release.yml`, Chrome Web Store submission support

## Releasing Changes

1. Make changes on `master`
2. Commit and optionally tag a specific version (e.g., `v1.0.1`)
3. Move the floating `v1` tag forward:
   ```bash
   git tag -f v1
   git push origin v1 --force
   ```
4. All consumers using `@v1` immediately pick up the change
