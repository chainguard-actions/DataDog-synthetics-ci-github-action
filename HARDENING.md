<!-- markdownlint-disable -->

# Hardening Report: DataDog--synthetics-ci-github-action/v3.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **DataDog--synthetics-ci-github-action/v3.8.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ }} expressions into shell commands, enabling script injection. Sub-rule (a) violations:

• bump-datadog-ci.yml: `VERSION=${{ github.event.inputs.datadog_ci_version }}` — attacker-controlled workflow_dispatch input interpolated directly into shell.
• bump-datadog-ci.yml: `run: git commit -a --message '[dep] Bump datadog-ci to \`${{ steps.bump-datadog-ci.outputs.VERSION }}\`'` — step output interpolated into shell.
• bump-datadog-ci.yml: `run: git push -u origin local-branch:bump-datadog-ci/${{ steps.bump-datadog-ci.outputs.VERSION }}` — step output interpolated into shell.
• release-version.yml: `yarn version ${{ github.event.inputs.semver }}` — attacker-controlled workflow_dispatch input interpolated directly into shell.
• release-version.yml: `git commit -m ${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated into shell.
• release-version.yml: `git push -u origin local-branch:release/${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated into shell.
• run-e2e-tests.yml: `echo 'Batch URL: ${{ steps.run-synthetics-tests.outputs.batch-url }}'` — step output interpolated into shell.

All these should be moved to env: variables and then referenced as quoted shell variables (e.g., "$VAR").

Locations:

- `.github/workflows/bump-datadog-ci.yml:42`
- `.github/workflows/bump-datadog-ci.yml:49`
- `.github/workflows/bump-datadog-ci.yml:51`
- `.github/workflows/release-version.yml:48`
- `.github/workflows/release-version.yml:58`
- `.github/workflows/release-version.yml:60`
- `.github/workflows/run-e2e-tests.yml:30`

### github-env-injection (severity: high)

In bump-datadog-ci.yml, the shell variable VERSION is assigned directly from the untrusted expression `${{ github.event.inputs.datadog_ci_version }}` (a workflow_dispatch input) and then written to $GITHUB_OUTPUT without sanitization: `echo "VERSION=$VERSION" >> $GITHUB_OUTPUT`. An attacker could inject newlines into the input to poison GITHUB_OUTPUT and set arbitrary environment variables or outputs for downstream steps. The required sanitization step (`printf '%s' "$VERSION" | tr -d '\n\r'`) is absent before the write.

Locations:

- `.github/workflows/bump-datadog-ci.yml:47`

### unpinned-uses (severity: high)

Every uses: reference across all workflow files is pinned to a mutable tag or version string rather than a full 40-character SHA commit hash. This exposes the workflows to supply-chain attacks if any referenced action's tag is moved or compromised. Failing references include:

• actions/create-github-app-token@v1
• actions/checkout@v4
• actions/setup-node@v4
• actions/github-script@v7
• actions/github-script@v6

All uses: references should be pinned to a full SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/bump-datadog-ci.yml:23`
- `.github/workflows/bump-datadog-ci.yml:27`
- `.github/workflows/bump-datadog-ci.yml:31`
- `.github/workflows/bump-datadog-ci.yml:55`
- `.github/workflows/bump-datadog-ci.yml:71`
- `.github/workflows/check-license.yml:11`
- `.github/workflows/check-release-version.yml:11`
- `.github/workflows/release-version-on-merge.yml:14`
- `.github/workflows/release-version-on-merge.yml:19`
- `.github/workflows/release-version.yml:27`
- `.github/workflows/release-version.yml:31`
- `.github/workflows/release-version.yml:35`
- `.github/workflows/release-version.yml:63`
- `.github/workflows/release-version.yml:75`
- `.github/workflows/release-version.yml:90`
- `.github/workflows/run-e2e-tests.yml:8`
- `.github/workflows/run-e2e-tests.yml:11`
- `.github/workflows/run-e2e-tests.yml:38`
- `.github/workflows/unit-tests.yml:12`
- `.github/workflows/unit-tests.yml:15`

### missing-permissions (severity: medium)

None of the workflow files define a top-level `permissions:` key, and no job within any workflow defines a job-level `permissions:` key. Without explicit permissions, workflows run with the default token permissions, which may be overly broad (e.g., write access to contents, pull-requests, etc.). Each workflow should declare the minimal permissions required, e.g. `permissions: read-all` at the top level and then grant specific write permissions only to the jobs that need them.

Locations:

- `.github/workflows/bump-datadog-ci.yml:1`
- `.github/workflows/check-license.yml:1`
- `.github/workflows/check-release-version.yml:1`
- `.github/workflows/release-version-on-merge.yml:1`
- `.github/workflows/release-version.yml:1`
- `.github/workflows/run-e2e-tests.yml:1`
- `.github/workflows/unit-tests.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all 4 findings across 7 workflow files. (1) unpinned-uses: Pinned all action references to full SHA hashes: actions/create-github-app-token@v1->d72941d..., actions/checkout@v4->11d5960..., actions/setup-node@v4->49933ea..., actions/github-script@v7->f28e40c..., actions/github-script@v6->d7906e4... (2) script-injection: Moved all ${{ }} expressions from run: blocks into env: blocks and referenced as plain shell variables. In bump-datadog-ci.yml: INPUT_VERSION env var replaces direct interpolation; BUMP_VERSION env var used for git commit/push. In release-version.yml: INPUT_SEMVER env var for yarn version; NEW_VERSION_TAG env var for git commit/push. In run-e2e-tests.yml: BATCH_URL env var replaces direct echo interpolation. JavaScript github-script steps now use process.env.VAR instead of ${{ }} interpolation. (3) github-env-injection: In bump-datadog-ci.yml, sanitized VERSION before writing to GITHUB_OUTPUT using safe=$(printf '%s' "$VERSION" | tr -d '\n\r') then writing $safe. (4) missing-permissions: Added top-level permissions: {} to all 7 workflow files. Added job-level permissions with minimum required: read-only jobs get contents: read; release/PR-creating jobs get contents: write and pull-requests: write.

### Iteration 2

**Fixes applied:** script-injection, script-injection

**Notes:**

Fixed two script injection findings:
1. hardened/action/.github/workflows/bump-datadog-ci.yml line 60: Added double quotes around `@datadog/datadog-ci@^$VERSION` in the yarn add command to prevent shell metacharacter injection from the user-controlled version input.
2. hardened/action/.github/workflows/run-e2e-tests.yml line 43: Added double quotes around `$RAW_RESULTS` in the echo command to prevent word splitting and glob expansion of the step output value.

