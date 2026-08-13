<!-- markdownlint-disable -->

# Hardening Report: DataDog--synthetics-ci-github-action/v4.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **DataDog--synthetics-ci-github-action/v4.1.1** was hardened automatically. 8 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks in bump-datadog-ci.yml directly interpolate ${{ }} expressions into shell commands (rule a — direct expression interpolation), allowing an attacker-controlled workflow_dispatch input to inject arbitrary shell commands.

- Line 48: `VERSION=${{ github.event.inputs.datadog_ci_version }}` — untrusted workflow_dispatch input interpolated directly into shell.
- Line 57: `run: git commit -a --message '[dep] Bump datadog-ci to \`${{ steps.bump-datadog-ci.outputs.VERSION }}\`'` — step output interpolated directly into shell.
- Line 60: `run: git push -u origin local-branch:bump-datadog-ci/${{ steps.bump-datadog-ci.outputs.VERSION }}` — step output interpolated directly into shell.

Locations:

- `.github/workflows/bump-datadog-ci.yml:48`
- `.github/workflows/bump-datadog-ci.yml:57`
- `.github/workflows/bump-datadog-ci.yml:60`

### script-injection (severity: high)

Multiple run: blocks in release-version.yml directly interpolate ${{ }} expressions into shell commands (rule a — direct expression interpolation).

- Line 53: `yarn version ${{ github.event.inputs.semver }}` — untrusted workflow_dispatch input interpolated directly into shell.
- Line 67: `git commit -m ${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated directly into shell.
- Line 70: `git push -u origin local-branch:release/${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated directly into shell.

Locations:

- `.github/workflows/release-version.yml:53`
- `.github/workflows/release-version.yml:67`
- `.github/workflows/release-version.yml:70`

### script-injection (severity: high)

The 'Example of using outputs' run: block in run-e2e-tests.yml has two violations:

(a) Line 34: `echo 'Batch URL: ${{ steps.run-synthetics-tests.outputs.batch-url }}'` — a ${{ }} expression is directly interpolated inside a run: shell command string.

(b) Line 36: `echo $RAW_RESULTS | jq '.'` — the shell variable $RAW_RESULTS (sourced from `${{ steps.run-synthetics-tests.outputs.raw-results }}` via the env: block) is expanded without double-quoting, allowing shell metacharacter injection.

Locations:

- `.github/workflows/run-e2e-tests.yml:34`
- `.github/workflows/run-e2e-tests.yml:36`

### github-env-injection (severity: high)

In bump-datadog-ci.yml, the shell variable VERSION is assigned directly from the untrusted workflow_dispatch input `${{ github.event.inputs.datadog_ci_version }}` (line 48) and then written to $GITHUB_OUTPUT on line 55 (`echo "VERSION=$VERSION" >> $GITHUB_OUTPUT`) without the required sanitization step (`printf '%s' "$VERSION" | tr -d '\n\r'`). A malicious input containing newlines could inject arbitrary key-value pairs into the GitHub output context.

Locations:

- `.github/workflows/bump-datadog-ci.yml:48`
- `.github/workflows/bump-datadog-ci.yml:55`

### permissions (severity: medium)

missing-permissions: The workflow file check-license.yml has no top-level `permissions:` key and its only job (check-license-3rdparty) also has no job-level `permissions:` key. This means the workflow runs with the default (potentially broad) repository permissions.

Locations:

- `.github/workflows/check-license.yml:1`

### permissions (severity: medium)

missing-permissions: The workflow file check-release-version.yml has no top-level `permissions:` key and its only job (check-release-version) also has no job-level `permissions:` key. This means the workflow runs with the default (potentially broad) repository permissions.

Locations:

- `.github/workflows/check-release-version.yml:1`

### permissions (severity: medium)

missing-permissions: The workflow file run-e2e-tests.yml has no top-level `permissions:` key and its only job (e2e) also has no job-level `permissions:` key. This means the workflow runs with the default (potentially broad) repository permissions.

Locations:

- `.github/workflows/run-e2e-tests.yml:1`

### permissions (severity: medium)

missing-permissions: The workflow file unit-tests.yml has no top-level `permissions:` key and its only job (build-and-test) also has no job-level `permissions:` key. This means the workflow runs with the default (potentially broad) repository permissions.

Locations:

- `.github/workflows/unit-tests.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, permissions

**Notes:**

Fixed all 8 findings across 5 workflow files:

1. bump-datadog-ci.yml: Moved ${{ github.event.inputs.datadog_ci_version }} to env block (DATADOG_CI_VERSION); moved ${{ steps.bump-datadog-ci.outputs.VERSION }} to env blocks (BUMP_VERSION) for commit and push steps; added newline sanitization (tr -d '\n\r') before writing VERSION to $GITHUB_OUTPUT.

2. release-version.yml: Moved ${{ github.event.inputs.semver }} to env block (SEMVER); moved ${{ steps.bump-version.outputs.NEW_VERSION_TAG }} to env blocks (NEW_VERSION_TAG) for commit and push steps.

3. run-e2e-tests.yml: Moved ${{ steps.run-synthetics-tests.outputs.batch-url }} to env block (BATCH_URL); double-quoted $RAW_RESULTS in echo command; added top-level permissions: {}.

4. check-license.yml: Added top-level permissions: {}.

5. check-release-version.yml: Added top-level permissions: {}.

6. unit-tests.yml: Added top-level permissions: {}.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in `.github/workflows/bump-datadog-ci.yml` at line 59: changed `yarn add @datadog/datadog-ci@^$VERSION` to `yarn add "@datadog/datadog-ci@^$VERSION"`. The `$VERSION` variable (derived from the `datadog_ci_version` workflow_dispatch input) is now double-quoted, preventing shell metacharacters from being interpreted as shell commands.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerabilities in three workflow files by moving all ${{ }} expression interpolations out of github-script blocks and into step-level env: blocks, then referencing them via process.env.VAR_NAME in the JavaScript:

1. release-version-on-merge.yml: Moved github.event.pull_request.head.ref to PR_HEAD_REF env var and github.event.pull_request.body to PR_BODY env var. Both accessed via process.env in the script.

2. bump-datadog-ci.yml: Moved steps.bump-datadog-ci.outputs.VERSION to BUMP_VERSION env var (used in create-pull-request step with template literals), and steps.create-pull-request.outputs.PULL_REQUEST_NUMBER to PULL_REQUEST_NUMBER env var (used with Number() conversion in create-comment step).

3. release-version.yml: Moved steps.bump-version.outputs.NEW_VERSION_TAG to NEW_VERSION_TAG env var in generate-release-notes step; moved NEW_VERSION_TAG, github.event.inputs.semver, and steps.generate-release-notes.outputs.RELEASE_NOTES to env vars in create-pull-request step; moved NEW_VERSION_TAG and steps.create-pull-request.outputs.PULL_REQUEST_NUMBER to env vars in create-comment step. All accessed via process.env in the JavaScript.

