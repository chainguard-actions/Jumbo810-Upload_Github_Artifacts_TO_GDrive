<!-- markdownlint-disable -->

# Hardening Report: Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **Jumbo810--Upload_Github_Artifacts_TO_GDrive/v2.3.1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All `uses:` references across all four workflow files are pinned to mutable tags or version strings rather than immutable 40-character SHA commit hashes. This exposes the workflows to supply-chain attacks if any referenced action is compromised or its tag is moved. Failing references:
- ci.yml: `actions/checkout@v4.1.1`, `actions/setup-node@v4.0.0`, `actions/upload-artifact@v4`
- release.yml: `actions/checkout@v4`, `actions/setup-node@v4`, `mindsers/changelog-reader-action@v2`, `softprops/action-gh-release@v1`
- test.yml: `actions/checkout@v4.1.1`, `actions/setup-node@v4.0.0`
- update-dist.yml: `actions/checkout@v4`, `actions/github-script@v7`

Locations:

- `.github/workflows/ci.yml:8`
- `.github/workflows/ci.yml:11`
- `.github/workflows/ci.yml:28`
- `.github/workflows/ci.yml:31`
- `.github/workflows/ci.yml:46`
- `.github/workflows/release.yml:9`
- `.github/workflows/release.yml:12`
- `.github/workflows/release.yml:29`
- `.github/workflows/release.yml:34`
- `.github/workflows/test.yml:12`
- `.github/workflows/test.yml:15`
- `.github/workflows/update-dist.yml:14`
- `.github/workflows/update-dist.yml:20`

### missing-permissions (severity: medium)

None of the four workflow files define a top-level `permissions:` block, and no job within any of these files defines job-level `permissions:` either. Without explicit permissions, workflows run with the default token permissions (which may be read/write depending on repository settings), violating the principle of least privilege.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

Sub-rule (a) violation: The `run:` block in the 'Check if upload was attempted' step of test.yml directly interpolates a GitHub Actions expression `${{ steps.test_upload.outcome }}` inside the shell command string. Before the shell executes the script, GitHub Actions performs template substitution, meaning any special characters in the value are passed raw to the shell. Offending line: `if [ "${{ steps.test_upload.outcome }}" == "success" ]; then`

Locations:

- `.github/workflows/test.yml:47`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection

**Notes:**

Fixed all three findings across four workflow files:

1. unpinned-uses: Pinned all 8 unique action references to full 40-char SHA hashes with original tags preserved as comments. Used lookup_action_sha to resolve each tag to its real commit SHA.

2. missing-permissions: Added top-level permissions blocks to all four workflows with minimal required permissions: ci.yml (contents: read), release.yml (contents: write for creating releases), test.yml (contents: read), update-dist.yml (contents: write + actions: read for pushing commits and downloading artifacts).

3. script-injection: Fixed the 'Check if upload was attempted' step in test.yml by moving `${{ steps.test_upload.outcome }}` into an env var `TEST_UPLOAD_OUTCOME` and referencing it as `$TEST_UPLOAD_OUTCOME` in the shell script, preventing template substitution from injecting raw values into the shell command.

