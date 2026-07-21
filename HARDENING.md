<!-- markdownlint-disable -->

# Hardening Report: DataDog--synthetics-ci-github-action/v3.7.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **DataDog--synthetics-ci-github-action/v3.7.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All `uses:` references across every workflow file use mutable tag refs (e.g. @v1, @v4, @v7, @v6) instead of immutable 40-character SHA digests. This exposes the workflows to supply-chain attacks if any referenced action's tag is moved or compromised. Failing references include: actions/create-github-app-token@v1, actions/checkout@v4, actions/setup-node@v4, actions/github-script@v7, actions/github-script@v6.

Locations:

- `.github/workflows/bump-datadog-ci.yml:24`
- `.github/workflows/bump-datadog-ci.yml:27`
- `.github/workflows/bump-datadog-ci.yml:31`
- `.github/workflows/bump-datadog-ci.yml:55`
- `.github/workflows/bump-datadog-ci.yml:75`
- `.github/workflows/check-license.yml:10`
- `.github/workflows/check-release-version.yml:10`
- `.github/workflows/release-version-on-merge.yml:16`
- `.github/workflows/release-version-on-merge.yml:20`
- `.github/workflows/release-version.yml:27`
- `.github/workflows/release-version.yml:31`
- `.github/workflows/release-version.yml:35`
- `.github/workflows/release-version.yml:68`
- `.github/workflows/release-version.yml:82`
- `.github/workflows/release-version.yml:100`
- `.github/workflows/run-e2e-tests.yml:8`
- `.github/workflows/run-e2e-tests.yml:11`
- `.github/workflows/run-e2e-tests.yml:36`
- `.github/workflows/unit-tests.yml:12`
- `.github/workflows/unit-tests.yml:15`

### script-injection (severity: high)

Multiple `run:` blocks directly interpolate `${{ ... }}` expressions into shell commands, enabling script injection. (a) bump-datadog-ci.yml: `VERSION=${{ github.event.inputs.datadog_ci_version }}` — attacker-controlled workflow_dispatch input interpolated directly into shell. (b) bump-datadog-ci.yml: `git commit -a --message '[dep] Bump datadog-ci to `${{ steps.bump-datadog-ci.outputs.VERSION }}`'` — step output interpolated into shell argument. (c) bump-datadog-ci.yml: `git push -u origin local-branch:bump-datadog-ci/${{ steps.bump-datadog-ci.outputs.VERSION }}` — step output interpolated into shell. (d) release-version.yml: `yarn version ${{ github.event.inputs.semver }}` — workflow_dispatch input interpolated directly into shell. (e) release-version.yml: `git commit -m ${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated unquoted into shell. (f) release-version.yml: `git push -u origin local-branch:release/${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` — step output interpolated into shell. (g) run-e2e-tests.yml: `echo 'Batch URL: ${{ steps.run-synthetics-tests.outputs.batch-url }}'` — step output interpolated into shell.

Locations:

- `.github/workflows/bump-datadog-ci.yml:43`
- `.github/workflows/bump-datadog-ci.yml:49`
- `.github/workflows/bump-datadog-ci.yml:51`
- `.github/workflows/release-version.yml:51`
- `.github/workflows/release-version.yml:60`
- `.github/workflows/release-version.yml:62`
- `.github/workflows/run-e2e-tests.yml:30`

### github-env-injection (severity: high)

In bump-datadog-ci.yml, the `Bump datadog-ci` step assigns the untrusted workflow_dispatch input `${{ github.event.inputs.datadog_ci_version }}` to the shell variable `VERSION`, then writes it to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`): `echo "VERSION=$VERSION" >> $GITHUB_OUTPUT`. An attacker-controlled version string containing newlines could inject arbitrary key-value pairs into the step output context.

Locations:

- `.github/workflows/bump-datadog-ci.yml:47`

### permissions (severity: medium)

None of the workflow files define a top-level `permissions:` block, and no job within any workflow defines job-level `permissions:`. Without explicit permissions, workflows run with the default (potentially broad) GITHUB_TOKEN permissions, violating the principle of least privilege. Affected files: bump-datadog-ci.yml, check-license.yml, check-release-version.yml, release-version-on-merge.yml, release-version.yml, run-e2e-tests.yml, unit-tests.yml.

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

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, permissions

**Notes:**

Fixed all 7 workflow files: (1) Pinned all uses: references to full SHA digests with tag comments. (2) Moved all ${{ }} expressions from run: blocks into env: blocks, referencing them as plain env vars in shell; in github-script steps, used process.env.* instead of inline interpolation. (3) Added printf/tr sanitization before writing to $GITHUB_OUTPUT in bump-datadog-ci.yml and release-version.yml. (4) Added permissions: {} to all 7 workflow files (bump-datadog-ci.yml, check-license.yml, check-release-version.yml, release-version-on-merge.yml, release-version.yml, run-e2e-tests.yml, unit-tests.yml).

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two unquoted shell variable expansions:
1. hardened/action/.github/workflows/run-e2e-tests.yml line 41: Changed `echo $RAW_RESULTS | jq '.'` to `echo "$RAW_RESULTS" | jq '.'` — quotes prevent shell metacharacters in the action output from being interpreted.
2. hardened/action/.github/workflows/bump-datadog-ci.yml line 50: Changed `yarn add @datadog/datadog-ci@^$VERSION` to `yarn add "@datadog/datadog-ci@^${VERSION}"` — quotes prevent word-splitting and glob expansion on the user-supplied version string.

