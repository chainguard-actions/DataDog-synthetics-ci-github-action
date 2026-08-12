<!-- markdownlint-disable -->

# Hardening Report: DataDog--synthetics-ci-github-action/v4.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **DataDog--synthetics-ci-github-action/v4.1.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ }} expressions into shell commands, violating rule (a). In bump-datadog-ci.yml: (1) line 48 sets VERSION directly from user-controlled input `VERSION=${{ github.event.inputs.datadog_ci_version }}`; (2) line 56 interpolates a step output into a git commit message `git commit -a --message '[dep] Bump datadog-ci to \`${{ steps.bump-datadog-ci.outputs.VERSION }}\`'`; (3) line 59 interpolates a step output into a git push refspec `git push -u origin local-branch:bump-datadog-ci/${{ steps.bump-datadog-ci.outputs.VERSION }}`. In release-version.yml: (4) line 55 interpolates a workflow_dispatch input directly into a shell command `yarn version ${{ github.event.inputs.semver }}`; (5) line 65 interpolates a step output into git commit `git commit -m ${{ steps.bump-version.outputs.NEW_VERSION_TAG }}`; (6) line 70 interpolates a step output into git push `git push -u origin local-branch:release/${{ steps.bump-version.outputs.NEW_VERSION_TAG }}`. In run-e2e-tests.yml: (7) line 33 interpolates a step output inside single-quoted echo `echo 'Batch URL: ${{ steps.run-synthetics-tests.outputs.batch-url }}'` (rule a — single quotes do not prevent GitHub Actions YAML template substitution); (8) line 35 uses unquoted shell variable `echo $RAW_RESULTS | jq '.'` where RAW_RESULTS holds a step output (rule b — unquoted expansion allows shell metacharacter injection).

Locations:

- `.github/workflows/bump-datadog-ci.yml:48`
- `.github/workflows/bump-datadog-ci.yml:56`
- `.github/workflows/bump-datadog-ci.yml:59`
- `.github/workflows/release-version.yml:55`
- `.github/workflows/release-version.yml:65`
- `.github/workflows/release-version.yml:70`
- `.github/workflows/run-e2e-tests.yml:33`
- `.github/workflows/run-e2e-tests.yml:35`

### github-env-injection (severity: high)

In bump-datadog-ci.yml, the run: block at the 'Bump datadog-ci' step (line 47) sets VERSION directly from the user-controlled workflow_dispatch input `VERSION=${{ github.event.inputs.datadog_ci_version }}` and then writes it to $GITHUB_OUTPUT on line 54 via `echo "VERSION=$VERSION" >> $GITHUB_OUTPUT` without the required sanitization step (`printf '%s' "$VERSION" | tr -d '\n\r'`). An attacker who can trigger the workflow_dispatch with a crafted version string containing newlines could inject arbitrary key=value pairs into GITHUB_OUTPUT, potentially overwriting subsequent step outputs.

Locations:

- `.github/workflows/bump-datadog-ci.yml:54`

### missing-permissions (severity: medium)

Four workflow files have no top-level `permissions:` key and no job-level `permissions:` key on any of their jobs. Without explicit permissions, workflows inherit the repository's default token permissions (which may be read/write for all scopes). Affected files: check-license.yml, check-release-version.yml, run-e2e-tests.yml, unit-tests.yml.

Locations:

- `.github/workflows/check-license.yml:1`
- `.github/workflows/check-release-version.yml:1`
- `.github/workflows/run-e2e-tests.yml:1`
- `.github/workflows/unit-tests.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all 8 script-injection locations across 3 workflow files by moving ${{ }} expressions into env: blocks and referencing them as plain shell variables. Fixed github-env-injection in bump-datadog-ci.yml by sanitizing VERSION with printf/tr before writing to GITHUB_OUTPUT. Added permissions: {} top-level block to 4 workflow files (check-license.yml, check-release-version.yml, run-e2e-tests.yml, unit-tests.yml) that lacked explicit permissions.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities:
1. release-version-on-merge.yml: Moved attacker-controlled `github.event.pull_request.head.ref` and `github.event.pull_request.body` from inline JavaScript template substitution into the step's `env:` block (as `PR_HEAD_REF` and `PR_BODY`). The github-script now reads them via `process.env.PR_HEAD_REF` and `process.env.PR_BODY`, eliminating the JavaScript injection vector.
2. bump-datadog-ci.yml: Quoted the `$VERSION` variable in `yarn add "@datadog/datadog-ci@^$VERSION"` to prevent shell metacharacter injection from the workflow_dispatch user input `datadog_ci_version`.

