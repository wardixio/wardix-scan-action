# Wardix Scan — GitHub Action

Security diff for mobile builds. Scans the APK/IPA your workflow just built and
**fails CI only on findings this build introduced** — known, persisting issues
never re-break the build, and dismissals made in the dashboard are honored.

Composite action wrapping the [`@wardix/cli` package](https://www.npmjs.com/package/@wardix/cli):
scan → sticky PR comment with the delta summary → optional SARIF upload to GitHub
code scanning → exit with the gate verdict.

## Usage

```yaml
name: wardix
on: pull_request

permissions:
  contents: read
  pull-requests: write     # sticky PR comment
  security-events: write   # only if upload-sarif: true

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./gradlew assembleRelease
      - uses: wardixio/wardix-scan-action@v1
        with:
          artifact: app/build/outputs/apk/release/app-release.apk
          key: ${{ secrets.WARDIX_KEY }}
          fail-on: high
          upload-sarif: "true"
```

Mint the key on your app space page in the [dashboard](https://app.wardix.io)
and store it as a repository secret — it is shown once and stored hashed.

## Inputs

| Input | Default | What it does |
|---|---|---|
| `artifact` | — (required) | Path to the built `.apk` or `.ipa`. |
| `key` | — (required) | Upload key (`wdx_ak_…`) — pass a secret, never a literal. |
| `fail-on` | `high` | Lowest severity of a NEW finding that fails the build (`critical` \| `high` \| `medium` \| `low` \| `info` \| `none`). |
| `comment` | `true` | Post and keep updating one sticky PR comment with the delta summary. |
| `upload-sarif` | `false` | Upload the SARIF log to GitHub code scanning (findings appear in the Security tab). |
| `timeout` | `1500` | Maximum seconds to wait for the scan. |
| `cli-version` | `0.2.3` | Exact `@wardix/cli` npm version passed to npx. |

`WARDIX_API_URL` / `WARDIX_APP_URL` env vars set at the job level pass
through to the CLI for self-hosted overrides.

## Outputs

| Output | What it is |
|---|---|
| `new-count` | Findings this build introduced. |
| `fixed-count` | Findings gone since the previous build. |
| `persisting-count` | Findings carried over (never gate). |
| `report-url` | Dashboard URL of the full scan report. |
| `gate` | `passed`, `failed` or `off` (`fail-on: none`). |

## Exit behavior

The scan step never fails the job directly, so the PR comment and SARIF upload
always run; the final Gate step re-raises the CLI exit code:

| Code | Meaning |
|---|---|
| 0 | Gate passed — nothing new at or above `fail-on`. |
| 1 | Gate failed — this build introduced findings at or above `fail-on`. |
| 2 | Scan errored, timed out, or completed without full required static coverage. |
| 3 | Bad configuration — missing/rejected key, unreadable artifact. |

## Code scanning notes

With `upload-sarif: "true"` the full findings snapshot (new + persisting) is
uploaded — GitHub diffs consecutive uploads itself, so alerts open and close
with your builds. Wardix's stable fingerprints ride along as
`partialFingerprints`, dashboard dismissals as SARIF suppressions, and every
alert deep-links back to the dashboard finding. Private repos need GitHub
Advanced Security for code scanning.

## Releases & versioning

This action lives at the root of its own public repo, `wardixio/wardix-scan-action`.
Reference it by the floating major tag — `v1` always points at the latest `v1.x`:

```yaml
- uses: wardixio/wardix-scan-action@v1
```

Pin to the current exact tag (`@v1.0.1`) for fully reproducible CI.

Maintainer release flow:

1. Push `action.yml` + `README.md` to the repo root.
2. Tag the release (for example, `git tag v1.0.1 && git push origin v1.0.1`).
3. Move the floating major tag to that exact release (`git tag -f v1 v1.0.1 && git push -f origin v1`).
4. (Optional) Create a GitHub Release and tick "Publish this Action to the GitHub Marketplace".
