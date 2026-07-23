<!-- markdownlint-disable -->

# Hardening Report: Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.4** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files use mutable tag-based `uses:` references instead of pinned 40-character SHA commit hashes, making them vulnerable to supply-chain attacks if the referenced action tags are moved or compromised.

Failing references:
- actions/checkout@v6 (ci.yml, manual-release.yml, release.yml, test.yml, update-dist.yml)
- actions/setup-node@v6.1.0 (ci.yml, manual-release.yml, release.yml, test.yml)
- actions/upload-artifact@v6 (ci.yml)
- mindsers/changelog-reader-action@v2 (manual-release.yml, release.yml, update-dist.yml)
- softprops/action-gh-release@v2 (manual-release.yml, release.yml, update-dist.yml)
- actions/github-script@v8 (update-dist.yml)

Locations:

- `.github/workflows/ci.yml:27`
- `.github/workflows/ci.yml:30`
- `.github/workflows/ci.yml:50`
- `.github/workflows/ci.yml:53`
- `.github/workflows/ci.yml:63`
- `.github/workflows/manual-release.yml:16`
- `.github/workflows/manual-release.yml:20`
- `.github/workflows/manual-release.yml:55`
- `.github/workflows/manual-release.yml:62`
- `.github/workflows/release.yml:13`
- `.github/workflows/release.yml:17`
- `.github/workflows/release.yml:37`
- `.github/workflows/release.yml:44`
- `.github/workflows/test.yml:13`
- `.github/workflows/test.yml:16`
- `.github/workflows/update-dist.yml:16`
- `.github/workflows/update-dist.yml:22`
- `.github/workflows/update-dist.yml:88`
- `.github/workflows/update-dist.yml:95`

### missing-permissions (severity: medium)

None of the workflow files define a top-level `permissions:` block, and no job within any workflow defines job-level `permissions:`. Without explicit permissions, workflows run with the default (often over-broad) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/manual-release.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

Multiple `run:` blocks directly interpolate `${{ ... }}` expressions into shell commands, allowing an attacker to inject arbitrary shell commands.

(a) `manual-release.yml` — `${{ inputs.version }}` is interpolated directly into several shell commands in the 'Commit Dist and Tag' step (e.g., `git tag ${{ inputs.version }}`, `git commit -m "...release ${{ inputs.version }}..."`) and in the 'Get version without v' step (`echo "VERSION=${{ inputs.version }}" | sed 's/v//' >> $GITHUB_OUTPUT`). A malicious `version` input value could inject shell metacharacters.

(a) `update-dist.yml` — `${{ github.event.workflow_run.head_commit.message }}` is interpolated directly into a shell variable assignment in the 'Commit and push changes' step: `COMMIT_MSG="${{ github.event.workflow_run.head_commit.message }}"`. A crafted commit message could break out of the double-quoted string and inject arbitrary commands.

(a) `test.yml` — `${{ steps.test_upload.outcome }}` is interpolated directly into a shell `if` condition: `if [ "${{ steps.test_upload.outcome }}" == "success" ]`.

Locations:

- `.github/workflows/manual-release.yml:43`
- `.github/workflows/manual-release.yml:47`
- `.github/workflows/manual-release.yml:48`
- `.github/workflows/manual-release.yml:50`
- `.github/workflows/manual-release.yml:54`
- `.github/workflows/update-dist.yml:72`
- `.github/workflows/test.yml:50`

### github-env-injection (severity: high)

Unsanitized untrusted values are written to special GitHub environment files without the required `printf '%s' ... | tr -d '\n\r'` sanitization step.

(b) `manual-release.yml` line 54 — The 'Get version without v' step writes `${{ inputs.version }}` directly to `$GITHUB_OUTPUT` via: `echo "VERSION=${{ inputs.version }}" | sed 's/v//' >> $GITHUB_OUTPUT`. The `inputs.version` value is not sanitized before the write; a newline embedded in the version string could inject additional key=value pairs into GITHUB_OUTPUT.

(e) `update-dist.yml` line 83 — The 'Commit and push changes' step writes `echo "RELEASE_VERSION=$VERSION" >> $GITHUB_ENV` where `$VERSION` is derived from `$COMMIT_MSG`, which is set from `${{ github.event.workflow_run.head_commit.message }}` (an attacker-controlled commit message). No sanitization (`tr -d '\n\r'`) is applied before the write, allowing newline injection into `$GITHUB_ENV`.

Locations:

- `.github/workflows/manual-release.yml:54`
- `.github/workflows/update-dist.yml:83`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all five workflow files:

**unpinned-uses**: Pinned all action references to full SHA hashes:
- actions/checkout@v6 → @d23441a48e516b6c34aea4fa41551a30e30af803
- actions/setup-node@v6.1.0 → @395ad3262231945c25e8478fd5baf05154b1d79f
- actions/upload-artifact@v6 → @b7c566a772e6b6bfb58ed0dc250532a479d7789f
- mindsers/changelog-reader-action@v2 → @1faaf50aa09d5793d9a100819973df801febfb31
- softprops/action-gh-release@v2 → @3bb12739c298aeb8a4eeaf626c5b8d85266b0e65
- actions/github-script@v8 → @ed597411d8f924073f98dfc5c65a23a2325f34cd

**missing-permissions**: Added top-level `permissions:` blocks to all five workflows. ci.yml and test.yml use `contents: read`; manual-release.yml, release.yml, and update-dist.yml use `contents: write` (required for pushing tags/commits and creating releases).

**script-injection**: Moved all `${{ }}` expressions out of `run:` blocks into `env:` blocks and referenced them as plain environment variables:
- manual-release.yml: `inputs.version` → `INPUT_VERSION` env var
- update-dist.yml: `github.event.workflow_run.head_commit.message` → `HEAD_COMMIT_MSG` env var
- test.yml: `steps.test_upload.outcome` → `TEST_OUTCOME` env var

**github-env-injection**: Added `printf '%s' ... | tr -d '\n\r'` sanitization before writing to `$GITHUB_OUTPUT` (manual-release.yml) and `$GITHUB_ENV` (update-dist.yml) to prevent newline injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the unquoted variable `$COMMIT_MSG` in the `[[` conditional test in `.github/workflows/update-dist.yml` at line 91. Changed `if [[ $COMMIT_MSG == release:\ v* ]]; then` to `if [[ "$COMMIT_MSG" == release:\ v* ]]; then`. This prevents an attacker-controlled commit message containing glob characters from altering the pattern matching behavior in the bash `[[` test.

### Iteration 3

**Fixes applied:** github-env-injection

**Notes:**

Fixed the github-env-injection finding in .github/workflows/release.yml at the 'Get version from tag' step. The original code wrote `${GITHUB_REF#refs/tags/v}` directly to $GITHUB_OUTPUT without sanitization. The fix uses `printf '%s' ... | tr -d '\n\r'` to strip newline and carriage-return characters from the tag-derived value before writing it to $GITHUB_OUTPUT, preventing an attacker from injecting arbitrary key-value pairs via a crafted tag name.

