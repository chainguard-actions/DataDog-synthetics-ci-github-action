<!-- markdownlint-disable -->

# Hardening Report: DataDog--synthetics-ci-github-action/v3.8.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **DataDog--synthetics-ci-github-action/v3.8.1** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): Multiple `run:` blocks directly interpolate `${{ ... }}` expressions into shell commands, enabling script injection. In bump-datadog-ci.yml: (1) `VERSION=${{ github.event.inputs.datadog_ci_version }}` injects user-controlled workflow_dispatch input directly into a shell variable assignment; (2) `git commit -a --message '[dep] Bump datadog-ci to \`${{ steps.bump-datadog-ci.outputs.VERSION }}\`'` interpolates a step output into a git commit message shell argument; (3) `git push -u origin local-branch:bump-datadog-ci/${{ steps.bump-datadog-ci.outputs.VERSION }}` interpolates a step output into a git push refspec. In release-version.yml: (4) `yarn version ${{ github.event.inputs.semver }}` injects a workflow_dispatch input directly into a shell command; (5) `git commit -m ${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` interpolates a step output unquoted into git commit; (6) `git push -u origin local-branch:release/${{ steps.bump-version.outputs.NEW_VERSION_TAG }}` interpolates a step output into a git push refspec. In run-e2e-tests.yml: (7) `echo 'Batch URL: ${{ steps.run-synthetics-tests.outputs.batch-url }}'` interpolates a step output into an echo command.

Locations:

- `.github/workflows/bump-datadog-ci.yml:44`
- `.github/workflows/bump-datadog-ci.yml:52`
- `.github/workflows/bump-datadog-ci.yml:54`
- `.github/workflows/release-version.yml:51`
- `.github/workflows/release-version.yml:59`
- `.github/workflows/release-version.yml:61`
- `.github/workflows/run-e2e-tests.yml:30`

### github-env-injection (severity: high)

In bump-datadog-ci.yml, the `Bump datadog-ci` step sets `VERSION=${{ github.event.inputs.datadog_ci_version }}` (a user-controlled workflow_dispatch input) and then writes it to `$GITHUB_OUTPUT` via `echo "VERSION=$VERSION" >> $GITHUB_OUTPUT` without applying the required sanitization (`printf '%s' ... | tr -d '\n\r'`) before the write. An attacker-supplied version string containing newlines could inject arbitrary key=value pairs into GITHUB_OUTPUT. Similarly, in release-version.yml, `NEW_VERSION_TAG` is derived from `yarn version ${{ github.event.inputs.semver }}` and written to `$GITHUB_OUTPUT` via `echo "NEW_VERSION_TAG=$NEW_VERSION_TAG" >> $GITHUB_OUTPUT` without sanitization.

Locations:

- `.github/workflows/bump-datadog-ci.yml:50`
- `.github/workflows/release-version.yml:57`

### missing-permissions (severity: medium)

None of the workflow files define a top-level `permissions:` key, and no job within any workflow defines job-level `permissions:`. This means all jobs run with the default (potentially broad) GITHUB_TOKEN permissions. Affected files: bump-datadog-ci.yml, check-license.yml, check-release-version.yml, release-version-on-merge.yml, release-version.yml, run-e2e-tests.yml, unit-tests.yml.

Locations:

- `.github/workflows/bump-datadog-ci.yml:1`
- `.github/workflows/check-license.yml:1`
- `.github/workflows/check-release-version.yml:1`
- `.github/workflows/release-version-on-merge.yml:1`
- `.github/workflows/release-version.yml:1`
- `.github/workflows/run-e2e-tests.yml:1`
- `.github/workflows/unit-tests.yml:1`

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tag refs instead of pinned 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved. Unpinned references found: `actions/create-github-app-token@v1`, `actions/checkout@v4`, `actions/setup-node@v4`, `actions/github-script@v7` (in bump-datadog-ci.yml and release-version.yml); `actions/create-github-app-token@v1`, `actions/checkout@v4`, `actions/github-script@v7` (in release-version-on-merge.yml); `actions/checkout@v4` (in check-license.yml and check-release-version.yml); `actions/checkout@v4`, `actions/setup-node@v4`, `actions/github-script@v6` (in run-e2e-tests.yml); `actions/checkout@v4`, `actions/setup-node@v4` (in unit-tests.yml).

Locations:

- `.github/workflows/bump-datadog-ci.yml:22`
- `.github/workflows/check-license.yml:11`
- `.github/workflows/check-release-version.yml:11`
- `.github/workflows/release-version-on-merge.yml:13`
- `.github/workflows/release-version.yml:26`
- `.github/workflows/run-e2e-tests.yml:7`
- `.github/workflows/unit-tests.yml:8`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, missing-permissions, unpinned-uses

**Notes:**

Fixed all 7 workflow files:

1. **script-injection** (7 locations): Moved all `${{ }}` expressions from `run:` shell commands into `env:` blocks, referencing them as plain environment variables. In github-script steps, moved expressions to `env:` and accessed via `process.env` in JavaScript.

2. **github-env-injection** (2 locations): Added `printf '%s' "$VAR" | tr -d '\n\r'` sanitization before writing user-controlled values to `$GITHUB_OUTPUT` in bump-datadog-ci.yml and release-version.yml.

3. **missing-permissions** (7 files): Added `permissions: {}` at top-level and `permissions: { contents: read }` at job-level for all workflow files.

4. **unpinned-uses** (all action references): Pinned all actions to full commit SHAs:
   - actions/create-github-app-token@v1 → @d72941d797fd3113feb6b93fd0dec494b13a2547
   - actions/checkout@v4 → @34e114876b0b11c390a56381ad16ebd13914f8d5
   - actions/setup-node@v4 → @49933ea5288caeca8642d1e84afbd3f7d6820020
   - actions/github-script@v7 → @f28e40c7f34bde8b3046d885e986cb6290c5673b
   - actions/github-script@v6 → @d7906e4ad0b1822421a7e6a35d5ca353c962f410

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two unquoted shell variable expansions:
1. `.github/workflows/run-e2e-tests.yml` line 38: Changed `echo $RAW_RESULTS | jq '.'` to `echo "$RAW_RESULTS" | jq '.'` to prevent shell metacharacter injection via the `steps.run-synthetics-tests.outputs.raw-results` value.
2. `.github/workflows/bump-datadog-ci.yml` line 48: Changed `yarn add @datadog/datadog-ci@^$VERSION` to `yarn add "@datadog/datadog-ci@^${VERSION}"` to prevent shell metacharacter injection via the `github.event.inputs.datadog_ci_version` value (which, while newline-sanitized, was not quoted against other metacharacters).

