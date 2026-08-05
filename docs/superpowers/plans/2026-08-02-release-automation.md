# Release Automation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a reviewed, manually initiated Release Please flow that automatically creates a GitHub release and then publishes `@pkg-nec/babel-plugin-istanbul` with npm provenance.

**Architecture:** Release Please manifest files establish `8.0.3` as the fork's release baseline. `prepare-release.yml` creates or refreshes the release PR only when manually dispatched; `create-github-release.yml` runs on each `main` push but releases only a merged pending Release Please PR. The renamed `release-npm-package.yml` continues to publish from a GitHub `release.published` event.

**Tech Stack:** GitHub Actions, `googleapis/release-please-action@v5.0.0`, Release Please manifest schema `v17.6.1`, Yarn 4, npm provenance.

## Global Constraints

- Use `googleapis/release-please-action@v5.0.0` in both new workflows.
- Use `$schema` pinned to `https://raw.githubusercontent.com/googleapis/release-please/v17.6.1/schemas/config.json`; do not use a branch URL.
- The package baseline is `8.0.3`; its bootstrap commit is `1afbfea332935477438b03c3029e69bedeeb8c4c`.
- The workflows are named `prepare-release.yml`, `create-github-release.yml`, and `release-npm-package.yml`.
- Both Release Please workflows create a fresh GitHub App installation token using the `RELEASE_PLEASE_APP_ID` repository variable and `RELEASE_PLEASE_APP_PRIVATE_KEY` repository secret, with explicit contents, issues, and pull-requests write permissions. Do not cache, reuse, or pass the token through an artifact, secret, or job output.
- Preserve `release-npm-package.yml` behavior exactly when renaming it.
- Do not edit generated `lib/` files or change package dependencies.
- Run Node, Corepack, and Yarn commands from Git Bash. Run `corepack enable` before `yarn install --immutable`.

---

## File structure

- Create: `release-please-config.json` — single-package Release Please manifest configuration, including the pinned schema and bootstrap SHA.
- Create: `.release-please-manifest.json` — records the established `8.0.3` release at repository root.
- Create: `.github/workflows/prepare-release.yml` — manually dispatched release-PR generation.
- Create: `.github/workflows/create-github-release.yml` — automated tag and GitHub Release creation after a release PR merge.
- Rename: `.github/workflows/release.yml` to `.github/workflows/release-npm-package.yml` — retains existing tested npm provenance publication behavior under an unambiguous name.
- Delete: `.github/workflows/release-please.yml` — superseded workflow that targets `master` and combines PR, release, and legacy token publication.

### Task 1: Add the Release Please baseline configuration

**Files:**

- Create: `release-please-config.json`
- Create: `.release-please-manifest.json`

**Interfaces:**

- Consumes: `package.json` at repository root, the `CHANGELOG.md` at repository root, and tag commit `1afbfea332935477438b03c3029e69bedeeb8c4c`.
- Produces: Release Please manifest state for the `.` package at version `8.0.3`, consumed by both GitHub workflows.

- [ ] **Step 1: Add the schema-pinned manifest configuration**

Create `release-please-config.json` with exactly:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/v17.6.1/schemas/config.json",
  "release-type": "node",
  "include-component-in-tag": false,
  "include-v-in-tag": true,
  "bootstrap-sha": "1afbfea332935477438b03c3029e69bedeeb8c4c",
  "packages": {
    ".": {}
  }
}
```

- [ ] **Step 2: Add the initial version manifest**

Create `.release-please-manifest.json` with exactly:

```json
{
  ".": "8.0.3"
}
```

- [ ] **Step 3: Validate both JSON files from Git Bash**

Run these commands separately from Git Bash at the repository root:

```bash
corepack enable
yarn install --immutable
yarn node --input-type=module -e "for (const file of ['release-please-config.json', '.release-please-manifest.json']) JSON.parse(await (await import('node:fs/promises')).readFile(file, 'utf8')); console.log('Release Please JSON OK')"
```

Expected: the command prints `Release Please JSON OK` and exits with status 0.

- [ ] **Step 4: Check the bootstrap reference**

Run:

```bash
git rev-parse v8.0.3
```

Expected: `1afbfea332935477438b03c3029e69bedeeb8c4c`, matching `bootstrap-sha`.

- [ ] **Step 5: Commit the manifest baseline**

```bash
git add release-please-config.json .release-please-manifest.json
git commit -m "ci: configure release please manifest"
```

### Task 2: Replace the old combined release workflow with split workflows

**Files:**

- Create: `.github/workflows/prepare-release.yml`
- Create: `.github/workflows/create-github-release.yml`
- Rename: `.github/workflows/release.yml` to `.github/workflows/release-npm-package.yml`
- Delete: `.github/workflows/release-please.yml`

**Interfaces:**

- Consumes: `RELEASE_PLEASE_APP_ID`, `RELEASE_PLEASE_APP_PRIVATE_KEY`, `release-please-config.json`, `.release-please-manifest.json`, and manual `workflow_dispatch` or a push to `main`.
- Produces: a pending Release Please PR in task 2A; then a `vX.Y.Z` GitHub Release in task 2B; finally the existing `release.published` event for `release-npm-package.yml`.

- [ ] **Step 1: Replace the old workflow with manual release-PR preparation**

Delete `.github/workflows/release-please.yml`. Create `.github/workflows/prepare-release.yml`:

```yaml
name: Prepare release

on:
  workflow_dispatch:

permissions:
  contents: write
  issues: write
  pull-requests: write

concurrency:
  group: release-please-main
  cancel-in-progress: false

jobs:
  prepare-release:
    name: Prepare release PR
    runs-on: ubuntu-latest
    steps:
      - name: Create Release Please GitHub App token
        id: release-please-app-token
        uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
        with:
          client-id: ${{ vars.RELEASE_PLEASE_APP_ID }}
          private-key: ${{ secrets.RELEASE_PLEASE_APP_PRIVATE_KEY }}
          permission-contents: write
          permission-issues: write
          permission-pull-requests: write
      - name: Create or update release PR
        uses: googleapis/release-please-action@v5.0.0
        with:
          token: ${{ steps.release-please-app-token.outputs.token }}
          target-branch: main
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
          skip-github-release: true
```

- [ ] **Step 2: Add the release-only workflow**

Create `.github/workflows/create-github-release.yml`:

```yaml
name: Create GitHub release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  issues: write
  pull-requests: write

concurrency:
  group: release-please-main
  cancel-in-progress: false

jobs:
  create-github-release:
    name: Create GitHub release
    runs-on: ubuntu-latest
    steps:
      - name: Create Release Please GitHub App token
        id: release-please-app-token
        uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
        with:
          client-id: ${{ vars.RELEASE_PLEASE_APP_ID }}
          private-key: ${{ secrets.RELEASE_PLEASE_APP_PRIVATE_KEY }}
          permission-contents: write
          permission-issues: write
          permission-pull-requests: write
      - name: Create tag and GitHub release for a merged release PR
        uses: googleapis/release-please-action@v5.0.0
        with:
          token: ${{ steps.release-please-app-token.outputs.token }}
          target-branch: main
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
          skip-github-pull-request: true
```

This job intentionally has no commit-message condition. On every `main` push, Release Please checks GitHub for a merged pending release PR. It no-ops for a normal `fix(deps)` PR and creates a tag/release only after the Release Please PR itself is merged.

- [ ] **Step 3: Rename the npm publication workflow without changing its content**

Run:

```bash
git mv .github/workflows/release.yml .github/workflows/release-npm-package.yml
```

Then verify its trigger remains:

```yaml
on:
  release:
    types: [published]
```

Verify the `npm publish --access public --provenance` step and the tag/package-version verification step are unchanged.

- [ ] **Step 4: Validate workflow syntax and release-file changes**

Run:

```bash
actionlint .github/workflows/prepare-release.yml .github/workflows/create-github-release.yml .github/workflows/release-npm-package.yml
git diff --check
git diff --name-status
```

Expected: `actionlint` and `git diff --check` exit with status 0. The name-status output shows the two new workflows, the npm-workflow rename, the old combined workflow deletion, and no unrelated files.

- [ ] **Step 5: Run the repository test gate from Git Bash**

Run these commands separately from Git Bash at the repository root:

```bash
corepack enable
yarn install --immutable
yarn test
```

Expected: StandardJS linting, Babel build, and NYC/Mocha unit tests all pass. Do not commit generated `lib/` output if the build changes it.

- [ ] **Step 6: Commit the workflow split**

```bash
git add -A -- .github/workflows
git commit -m "ci: split release please workflows"
```

### Task 3: Verify the live release lifecycle after merge

**Files:**

- Verify: `release-please-config.json`
- Verify: `.release-please-manifest.json`
- Verify: `.github/workflows/prepare-release.yml`
- Verify: `.github/workflows/create-github-release.yml`
- Verify: `.github/workflows/release-npm-package.yml`

**Interfaces:**

- Consumes: the `RELEASE_PLEASE_APP_ID` repository variable, the `RELEASE_PLEASE_APP_PRIVATE_KEY` repository secret, a manually dispatched workflow, a conventional `fix(deps): ...` merge to `main`, and GitHub Release events.
- Produces: evidence that routine work does not release immediately and that a reviewed release PR triggers exactly one provenance npm publication.

- [ ] **Step 1: Configure the release credential before dispatching the workflow**

Create a dedicated organization-owned GitHub App, leave webhook events inactive, and install it only on this repository. Grant Contents, Pull requests, and Issues write permissions. Store its client ID in the `RELEASE_PLEASE_APP_ID` repository Actions variable and its PEM private key in the `RELEASE_PLEASE_APP_PRIVATE_KEY` repository Actions secret. Generate a fresh installation token in each Release Please job with explicit permission inputs; do not cache, reuse, or expose the token in an artifact, secret, job output, or workflow log.

- [ ] **Step 2: Verify that normal work does not publish**

After a normal `fix(deps): ...` PR is merged to `main`, inspect the `Create GitHub release` workflow run. Confirm it completes without a tag or GitHub Release, because no Release Please PR has been merged.

- [ ] **Step 3: Exercise the manual review gate**

From GitHub Actions, manually dispatch `Prepare release` on `main`. Confirm it opens a Release Please PR targeting `main` with a patch version, updates `package.json`, `CHANGELOG.md`, and `.release-please-manifest.json`, and has the pending Release Please status label.

- [ ] **Step 4: Exercise the automatic release and publish chain**

Review and merge the generated Release Please PR. Confirm that `Create GitHub release` creates exactly one `vX.Y.Z` tag and published GitHub Release. Then confirm `Publish to npm` from `release-npm-package.yml` starts from that release event, passes its tag/version check and `yarn test`, and publishes with npm provenance.

- [ ] **Step 5: Record the completed production verification**

Add the GitHub Release URL and the npm package version to the release PR discussion or release notes. Do not make an additional repository commit solely for this operational record.
