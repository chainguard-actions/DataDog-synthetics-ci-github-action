<!-- markdownlint-disable -->

# Hardening Report: DataDog--synthetics-ci-github-action/v3.8.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **DataDog--synthetics-ci-github-action/v3.8.2** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `run:` blocks interpolate `${{ ... }}` expressions directly into shell commands, allowing an attacker to inject arbitrary shell code.

1. `.github/workflows/bump-datadog-ci.yml` line 47: `VERSION=${{ github.event.inputs.datadog_ci_version }}` — user-controlled workflow_dispatch input interpolated directly into shell.
2. `.github/workflows/bump-datadog-ci.yml` line 55: `run: git commit -a --message '[dep] Bump datadog-ci to \`${{ steps.bump-datadog-ci.outputs.VERSION }}\`'` — step output interpolated into a shell command.
3. `.github/workflows/bump-datadog-ci.yml` line 58: `run: git push -u origin local-branch:bump-datadog-ci/${{ steps.bump-datadog-ci.outputs.VERSION }}` — step output interpolated into a shell command.
4. `.github/workflows/release-version.yml` line 53: `yarn version ${{ github.event.inputs.semver }}` — user-controlled workflow_dispatch input interpolated directly into shell.
5. `.github/workflows/release-version.yml` line 62: `git commit -m ${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated into a shell command.
6. `.github/workflows/release-version.yml` line 65: `git push -u origin local-branch:release/${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated into a shell command.
7. `.github/workflows/run-e2e-tests.yml` line 33: `echo 'Batch URL: ${{ steps.run-synthetics-tests.outputs.batch-url }}'` — step output interpolated directly into shell.

Sub-rule (b): `.github/workflows/run-e2e-tests.yml` line 35: `echo $RAW_RESULTS | jq '.'` — the env var `RAW_RESULTS` (sourced from `${{ steps.run-synthetics-tests.outputs.raw-results }}`) is expanded unquoted, allowing shell metacharacter injection.

Locations:

- `.github/workflows/bump-datadog-ci.yml:47`
- `.github/workflows/bump-datadog-ci.yml:55`
- `.github/workflows/bump-datadog-ci.yml:58`
- `.github/workflows/release-version.yml:53`
- `.github/workflows/release-version.yml:62`
- `.github/workflows/release-version.yml:65`
- `.github/workflows/run-e2e-tests.yml:33`
- `.github/workflows/run-e2e-tests.yml:35`

### github-env-injection (severity: high)

A `run:` block writes a value derived from an untrusted input to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

In `.github/workflows/bump-datadog-ci.yml`, the `Bump datadog-ci` step assigns the user-controlled `workflow_dispatch` input `github.event.inputs.datadog_ci_version` to the shell variable `VERSION` (line 47), then writes it directly to `$GITHUB_OUTPUT` (line 53): `echo "VERSION=$VERSION" >> $GITHUB_OUTPUT`. A newline embedded in the input value could inject additional key=value pairs into the output file.

Locations:

- `.github/workflows/bump-datadog-ci.yml:53`

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions by mutable version tags instead of full 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved.

Failing references:
- `actions/checkout@v4` (bump-datadog-ci.yml, check-license.yml, check-release-version.yml, release-version.yml, run-e2e-tests.yml, unit-tests.yml)
- `actions/setup-node@v4` (bump-datadog-ci.yml, release-version.yml, run-e2e-tests.yml, unit-tests.yml)
- `actions/github-script@v7` (bump-datadog-ci.yml lines 62 and 71; release-version.yml lines 68, 78, 92)
- `actions/github-script@v6` (run-e2e-tests.yml line 37)

Locations:

- `.github/workflows/bump-datadog-ci.yml:29`
- `.github/workflows/bump-datadog-ci.yml:32`
- `.github/workflows/bump-datadog-ci.yml:62`
- `.github/workflows/bump-datadog-ci.yml:71`
- `.github/workflows/check-license.yml:12`
- `.github/workflows/check-release-version.yml:12`
- `.github/workflows/release-version.yml:32`
- `.github/workflows/release-version.yml:35`
- `.github/workflows/release-version.yml:68`
- `.github/workflows/release-version.yml:78`
- `.github/workflows/release-version.yml:92`
- `.github/workflows/run-e2e-tests.yml:9`
- `.github/workflows/run-e2e-tests.yml:11`
- `.github/workflows/run-e2e-tests.yml:37`
- `.github/workflows/unit-tests.yml:12`
- `.github/workflows/unit-tests.yml:14`

### missing-permissions (severity: medium)

Four workflow files have no top-level `permissions:` key and no job-level `permissions:` key on any of their jobs. Without explicit permissions, workflows inherit the repository's default token permissions (often `write-all`), granting unnecessary access.

- `check-license.yml`: no permissions declared
- `check-release-version.yml`: no permissions declared
- `unit-tests.yml`: no permissions declared
- `run-e2e-tests.yml`: no permissions declared

Locations:

- `.github/workflows/check-license.yml:1`
- `.github/workflows/check-release-version.yml:1`
- `.github/workflows/unit-tests.yml:1`
- `.github/workflows/run-e2e-tests.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all 4 finding types across 6 workflow files:

1. script-injection: Moved all ${{ }} expressions from run: blocks into env: blocks. Shell commands now use plain env vars (${VAR}). For github-script steps, expressions moved to env: and accessed via process.env.* in JS. Covers all 8 locations (bump-datadog-ci.yml lines 47,55,58; release-version.yml lines 53,62,65; run-e2e-tests.yml lines 33,35).

2. github-env-injection: Added printf '%s' ... | tr -d '\n\r' sanitization before writing VERSION and NEW_VERSION_TAG to $GITHUB_OUTPUT in bump-datadog-ci.yml and release-version.yml.

3. unpinned-uses: Pinned all mutable tags to full SHAs with tag comments: actions/checkout@v4→34e114876b0b11c390a56381ad16ebd13914f8d5, actions/setup-node@v4→49933ea5288caeca8642d1e84afbd3f7d6820020, actions/github-script@v7→f28e40c7f34bde8b3046d885e986cb6290c5673b, actions/github-script@v6→d7906e4ad0b1822421a7e6a35d5ca353c962f410. Applied across bump-datadog-ci.yml, check-license.yml, check-release-version.yml, release-version.yml, run-e2e-tests.yml, unit-tests.yml.

4. missing-permissions: Added top-level 'permissions: contents: read' to check-license.yml, check-release-version.yml, unit-tests.yml, and run-e2e-tests.yml.

### Iteration 2

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed release-version-on-merge.yml: (1) Pinned `actions/github-script@v7` to its full commit SHA `f28e40c7f34bde8b3046d885e986cb6290c5673b` with a `# v7` comment for readability. (2) Moved `github.event.pull_request.head.ref` and `github.event.pull_request.body` out of the `script:` JavaScript block into an `env:` block as `PR_HEAD_REF` and `PR_BODY`, then referenced them via `process.env.PR_HEAD_REF` and `process.env.PR_BODY` inside the script to prevent script injection.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in .github/workflows/bump-datadog-ci.yml at line 50. Changed `yarn add @datadog/datadog-ci@^$VERSION` to `yarn add "@datadog/datadog-ci@^${VERSION}"` to properly quote the shell variable, preventing attacker-supplied version strings containing shell metacharacters from being interpreted by the shell.

