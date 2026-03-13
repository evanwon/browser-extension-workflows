# Browser Extension Workflows

Shared CI/CD workflows and versioning tools for browser extensions. Provides two things:

1. **Reusable GitHub Actions workflows** for PR validation and build/release (test, lint, build, sign, AMO submission, GitHub releases)
2. **CLI tools** for version management (`extension-version-bump`, `extension-validate-versions`)

Supports Firefox and Chrome. Chrome support is opt-in via `chrome-enabled: true` — repos that don't set it are completely unaffected.

## Getting Started

### 1. Install the package

```bash
npm install github:evanwon/browser-extension-workflows --save-dev
```

### 2. Add npm scripts to your `package.json`

```json
{
  "scripts": {
    "version:bump": "extension-version-bump",
    "version:check": "extension-validate-versions",
    "test:ci": "jest --ci --coverage --maxWorkers=2"
  }
}
```

The `test:ci` script is called by the shared workflows — use whatever test runner you prefer, but it must exist.

### 3. Create workflow files

The shared workflows are [reusable workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows). Your repo needs thin caller files in `.github/workflows/` that delegate to them.

#### `.github/workflows/test-pr.yml`

```yaml
name: Test Pull Request

on:
  pull_request:
    branches: [main]

jobs:
  test:
    uses: evanwon/browser-extension-workflows/.github/workflows/test-pr.yml@v1
    with:
      lint-warnings-as-errors: false
      validate-versions: true
      security-audit-strict: false
      upload-coverage: false
      # chrome-enabled: true  # Uncomment to enable Chrome build validation
```

#### `.github/workflows/build-release.yml`

```yaml
name: Build and Release

on:
  push:
    tags:
      - 'v*'

  workflow_dispatch:
    inputs:
      create_release:
        description: 'Create GitHub release'
        type: boolean
        default: false
      submit_to_amo:
        description: 'Submit to AMO'
        type: boolean
        default: false
      channel:
        description: 'AMO channel'
        type: choice
        options:
          - listed
          - unlisted
        default: listed

permissions:
  contents: write

jobs:
  build:
    uses: evanwon/browser-extension-workflows/.github/workflows/build-release.yml@v1
    with:
      extension-name: my-extension
      extension-display-name: "My Extension"
      extension-description: "One-line description of your extension."
      manifest-version: 3
      lint-warnings-as-errors: false
      validate-tag-matches-manifest: true
      amo-submission-enabled: ${{ vars.AMO_SUBMISSION_ENABLED == 'true' }}
      issues-url: "https://github.com/you/my-extension/issues"
      amo-url: "https://addons.mozilla.org/en-US/firefox/addon/my-extension/"
      firefox-min-version: "109+"
      permissions-description: "activeTab, storage"
      create-release: ${{ github.event.inputs.create_release == 'true' }}
      submit-to-amo: ${{ github.event.inputs.submit_to_amo == 'true' }}
      channel: ${{ github.event.inputs.channel || 'listed' }}
      # Chrome support (opt-in):
      # chrome-enabled: true
      # cws-submission-enabled: ${{ vars.CWS_SUBMISSION_ENABLED == 'true' }}
      # cws-extension-id: ${{ vars.CWS_EXTENSION_ID }}
      # chrome-min-version: "102+"
    secrets: inherit
```

### 4. Set up your manifest

Your `src/manifest.json` must include both `version` and `version_name` fields:

```json
{
  "version": "1.0.0",
  "version_name": "1.0.0"
}
```

The `version:bump` tool manages both fields automatically.

### 5. Configure GitHub secrets (optional)

For automated AMO signing/submission and/or Chrome Web Store uploads:

| Type | Name | Description |
|------|------|-------------|
| Secret | `AMO_API_KEY` | AMO API issuer ID (Firefox) |
| Secret | `AMO_API_SECRET` | AMO API secret (Firefox) |
| Variable | `AMO_SUBMISSION_ENABLED` | Set to `true` to auto-submit stable tags to AMO listed channel |
| Secret | `CWS_CLIENT_ID` | Chrome Web Store API client ID |
| Secret | `CWS_CLIENT_SECRET` | Chrome Web Store API client secret |
| Secret | `CWS_REFRESH_TOKEN` | Chrome Web Store API refresh token |
| Variable | `CWS_SUBMISSION_ENABLED` | Set to `true` to enable CWS uploads on stable tags |
| Variable | `CWS_EXTENSION_ID` | Chrome Web Store extension ID |

Without credentials, builds still run — you just get unsigned artifacts.

## Conventions

This package is opinionated. It assumes:

- **Firefox and Chrome** — Firefox uses `web-ext` and AMO signing; Chrome build/validation/CWS submission is opt-in via `chrome-enabled: true`
- **Source in `src/`** — your extension source lives in `src/`, including `src/manifest.json`
- **Manifest V2 or V3** — configurable via `manifest-version` input (default: 3)
- **`npm ci --ignore-scripts`** — dependencies are installed without running lifecycle scripts (security)
- **Node 22** — workflows use Node.js 22
- **Tag-driven releases** — pushing `v*` tags triggers builds

## CLI Tools

### `extension-version-bump`

Bumps versions in `src/manifest.json` and `package.json`, then commits and tags.

```bash
# From stable state
npm run version:bump patch          # 1.0.0 -> 1.0.1
npm run version:bump minor          # 1.0.0 -> 1.1.0
npm run version:bump major          # 1.0.0 -> 2.0.0

# Start a pre-release series
npm run version:bump rc patch       # 1.0.0 -> 1.0.1-rc1
npm run version:bump rc minor       # 1.0.0 -> 1.1.0-rc1
npm run version:bump beta major     # 1.0.0 -> 2.0.0-beta1

# From pre-release state
npm run version:bump rc             # 1.1.0-rc1 -> 1.1.0-rc2
npm run version:bump stable         # 1.1.0-rc2 -> 1.1.0

# Escape hatch: set an explicit version
npm run version:bump 2.0.0-rc1

# Options
npm run version:bump -- --dry-run   # Preview without writing files
npm run version:bump -- --no-git    # Update files only, skip commit and tag
```

After bumping, push the tag to trigger the build workflow:

```bash
git push origin v1.1.0
```

### `extension-validate-versions`

Validates that `src/manifest.json` and `package.json` have consistent versions.

```bash
npm run version:check
```

Checks performed:
- `package.json` version matches `manifest.json` version
- `manifest.json` has a `version_name` field
- Pre-release: `version` is 4 segments with correct suffix encoding
- Stable: `version` equals `version_name`, both `X.Y.Z`
- Chrome template (if `manifests/chrome/manifest.json` exists): versions match Firefox manifest

## Version Encoding

Firefox requires numeric-only `version` fields, so pre-release versions use a 4-segment encoding:

| State | `version` | `version_name` | Notes |
|-------|-----------|-----------------|-------|
| Stable `1.0.0` | `1.0.0` | `1.0.0` | Both fields match |
| Pre-release `1.1.0-rc1` | `1.0.0.1` | `1.1.0-rc1` | `version` uses *previous* stable as base |
| Pre-release `1.1.0-rc2` | `1.0.0.2` | `1.1.0-rc2` | 4th segment = suffix number |
| Promoted `1.1.0` | `1.1.0` | `1.1.0` | Back to 3 segments |

The key insight: the 4-segment `version` uses the **previous stable** as its base (not the target), so Firefox correctly sorts `1.0.0.1` < `1.1.0`. The `version_name` field carries the human-readable version; Firefox warns about it but does not use it for sorting.

Note: `package.json` `version` is kept in sync with `manifest.json` `version` (the numeric form). During pre-release, this means `package.json` will contain a 4-segment version like `1.0.0.1`, not the semver-style `1.1.0-rc1`. This is expected — `version_name` in the manifest is the human-readable form.

## Workflow Reference

### test-pr.yml

Runs on PRs. Steps: checkout, install, validate versions, security audit, web-ext lint, run tests, upload coverage.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `lint-warnings-as-errors` | boolean | `false` | Pass `--warnings-as-errors` to `web-ext lint` |
| `validate-versions` | boolean | `true` | Run `extension-validate-versions` |
| `security-audit-strict` | boolean | `false` | Fail on `npm audit` findings (vs. warn-only) |
| `upload-coverage` | boolean | `false` | Upload coverage to Codecov |
| `chrome-enabled` | boolean | `false` | Enable Chrome build validation in PR checks |

### build-release.yml

Runs on `v*` tags or manual dispatch. Steps: checkout, install, validate, audit, test, lint, build, determine AMO channel, sign/submit, generate changelog, create GitHub release.

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `extension-name` | string | yes | — | Artifact slug (e.g., `my-extension`) |
| `extension-display-name` | string | yes | — | Human name for release titles |
| `extension-description` | string | yes | — | One-line description for release notes |
| `manifest-version` | number | no | `3` | Expected `manifest_version` (2 or 3) |
| `lint-warnings-as-errors` | boolean | no | `false` | `--warnings-as-errors` for lint |
| `validate-tag-matches-manifest` | boolean | no | `true` | Verify git tag matches `version_name` |
| `amo-submission-enabled` | boolean | no | `false` | Whether AMO listed submission is enabled |
| `issues-url` | string | yes | — | GitHub issues URL (used in pre-release notes) |
| `amo-url` | string | no | `''` | AMO listing URL (empty = omit from notes) |
| `firefox-min-version` | string | no | `'109+'` | Minimum Firefox version (release notes footer) |
| `permissions-description` | string | yes | — | Permissions list (release notes footer) |
| `create-release` | boolean | no | `false` | Create GitHub release (for `workflow_dispatch`) |
| `submit-to-amo` | boolean | no | `false` | Submit to AMO (for `workflow_dispatch`) |
| `channel` | string | no | `'listed'` | AMO channel (for `workflow_dispatch`) |
| `version-notes` | string | no | `''` | Optional AMO submission notes |
| `chrome-enabled` | boolean | no | `false` | Enable Chrome build, validation, and packaging |
| `chrome-build-dir` | string | no | `'build/chrome'` | Where Chrome build output lives |
| `cws-submission-enabled` | boolean | no | `false` | Enable Chrome Web Store submission |
| `cws-extension-id` | string | no | `''` | Chrome Web Store extension ID |
| `chrome-min-version` | string | no | `'102+'` | Minimum Chrome version (release notes footer) |

**Secrets** (all optional):
- `AMO_API_KEY` — AMO API issuer ID
- `AMO_API_SECRET` — AMO API secret
- `CWS_CLIENT_ID` — Chrome Web Store API client ID
- `CWS_CLIENT_SECRET` — Chrome Web Store API client secret
- `CWS_REFRESH_TOKEN` — Chrome Web Store API refresh token

**Release behavior:**
- **Stable tag push** (`v1.0.0`) — test, build, sign, create GitHub release. If `amo-submission-enabled` is true, submits to AMO. If `chrome-enabled` and `cws-submission-enabled`, submits to CWS
- **Pre-release tag push** (`v1.0.0-rc1`) — test, build, sign unlisted, create GitHub pre-release. Never submitted to AMO/CWS listed
- **Manual dispatch** — test, build. Optionally create release and/or submit to stores based on inputs
- When `chrome-enabled` is true, releases include both Firefox `.xpi` and Chrome `.zip` assets

## Versioning This Repo

This repo uses a **floating major-version tag** (`v1`) following the GitHub Actions convention:

- Callers pin to `@v1` and automatically get patch fixes
- Callers can pin to `@v1.0.0` for a locked version
- When releasing a patch, move the `v1` tag forward to the new commit

## License

MIT
