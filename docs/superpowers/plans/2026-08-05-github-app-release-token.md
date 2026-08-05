# GitHub App Release Token Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the long-lived Release Please PAT with a short-lived,
repository-scoped GitHub App token and provide a reusable npm release runbook.

**Architecture:** A dedicated organization-owned GitHub App is installed on
only this repository. Each Release Please workflow creates an installation
token in its own job with `actions/create-github-app-token`, then passes that
ephemeral output directly to Release Please. A generic documentation file
explains the same model without repository-specific content, while
`RELEASING.md` keeps this repository's local policy.

**Tech Stack:** GitHub Apps, GitHub Actions,
`actions/create-github-app-token@v3.2.0`,
`googleapis/release-please-action@v5.0.0`, Release Please, npm provenance.

## Global Constraints

- Create one organization-owned GitHub App per adopting repository; each App
  has a separate client ID and private key and is installed on that repository
  only.
- Grant the App only Contents, Pull requests, and Issues read/write
  repository permissions; do not subscribe it to webhook events.
- Store the App client ID in `RELEASE_PLEASE_APP_ID` as a repository Actions
  variable and its PEM private key in `RELEASE_PLEASE_APP_PRIVATE_KEY` as a
  repository Actions secret.
- Use `actions/create-github-app-token` at
  `bcd2ba49218906704ab6c1aa796996da409d3eb1` (v3.2.0) in both Release Please
  jobs.
- Explicitly request Contents, Pull requests, and Issues write permissions
  from the token action. Do not set cross-repository targeting inputs.
- Pass `steps.release-please-app-token.outputs.token` directly to Release
  Please. Never persist, cache, artifact, expose as a job output, or log this
  token. Do not set `skip-token-revoke`.
- Keep the existing Release Please action revision and the reviewed release PR
  → GitHub Release → npm provenance-publication lifecycle unchanged.
- Do not change generated `lib/` output or package dependencies.
- The generic release guide must contain no organization, repository, package,
  historical PR, release-version, or repository-specific URL references.
- Do not delete `RELEASE_PLEASE_TOKEN` until a live App-authenticated release
  preparation and release chain have succeeded.

---

## File structure

- Modify: `.github/workflows/prepare-release.yml` — mint a token for the
  manually dispatched Release Please PR job.
- Modify: `.github/workflows/create-github-release.yml` — mint a token for
  the release-only job after a merged Release Please PR.
- Create: `docs/automated-npm-release.md` — copy-ready, generic setup and
  operations runbook for npm release automation.
- Modify: `RELEASING.md` — link repository policy to the generic runbook.
- Modify: `docs/superpowers/specs/2026-08-02-release-automation-design.md` —
  replace superseded PAT guidance with the GitHub App model.
- Modify: `docs/superpowers/plans/2026-08-02-release-automation.md` — replace
  superseded PAT setup and validation instructions.

### Task 1: Provision the repository-specific GitHub App

**Files:**

- No repository files change in this task.

**Interfaces:**

- Consumes: organization ownership of the repository and access to its GitHub
  App and Actions settings.
- Produces: a selected-repository App installation, the client-ID variable,
  and the private-key secret consumed by both workflow files in Task 2.

- [ ] **Step 1: Create the organization-owned App**

  In the organization's GitHub App settings, create an App whose name makes
  its dedicated repository ownership clear. Set its owner to the organization,
  leave webhook events inactive, and configure these repository permissions:

  | Permission | Access |
  | --- | --- |
  | Contents | Read and write |
  | Pull requests | Read and write |
  | Issues | Read and write |

  Do not grant organization permissions, Actions permissions, or access to
  other repositories.

- [ ] **Step 2: Install the App with repository-only scope**

  Install the App for the organization using **Only select repositories**, then
  select `babel-plugin-istanbul` and no other repository. Record the generated
  App client ID without putting it in a tracked file.

- [ ] **Step 3: Generate and store the App credentials**

  Generate a PEM private key in the App settings. In this repository's Actions
  settings, create the following values:

  | GitHub Actions setting | Name | Value |
  | --- | --- | --- |
  | Variable | `RELEASE_PLEASE_APP_ID` | the App client ID |
  | Secret | `RELEASE_PLEASE_APP_PRIVATE_KEY` | the complete PEM private key |

  Confirm the key is stored only as a secret and never copied into a shell
  command, workflow file, issue, pull request, artifact, or cache.

- [ ] **Step 4: Record the App recovery procedure**

  In the organization's credential inventory, record the App name, its
  selected-repository installation, and the owner responsible for key
  rotation. Do not record the PEM material or an installation token.

### Task 2: Generate a token inside each Release Please job

**Files:**

- Modify: `.github/workflows/prepare-release.yml`
- Modify: `.github/workflows/create-github-release.yml`

**Interfaces:**

- Consumes: `vars.RELEASE_PLEASE_APP_ID` and
  `secrets.RELEASE_PLEASE_APP_PRIVATE_KEY` from Task 1.
- Produces: `steps.release-please-app-token.outputs.token`, scoped to the
  current repository for the immediately following Release Please action.

- [ ] **Step 1: Add the token-minting step to the preparation workflow**

  In `prepare-release.yml`, insert this step immediately before `Create or
  update release PR`:

  ```yaml
      - name: Create Release Please GitHub App token
        id: release-please-app-token
        uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
        with:
          client-id: ${{ vars.RELEASE_PLEASE_APP_ID }}
          private-key: ${{ secrets.RELEASE_PLEASE_APP_PRIVATE_KEY }}
          permission-contents: write
          permission-issues: write
          permission-pull-requests: write
  ```

  Do not add `owner`, `repositories`, `enterprise`, or `skip-token-revoke`.
  With no targeting inputs, the action scopes its token to the current
  repository's installation.

- [ ] **Step 2: Wire the preparation action to the generated token**

  Change only the `token` input of `Create or update release PR`:

  ```yaml
          token: ${{ steps.release-please-app-token.outputs.token }}
  ```

  Keep `target-branch`, `config-file`, `manifest-file`, and
  `skip-github-release` unchanged.

- [ ] **Step 3: Add the corresponding token step to the release workflow**

  In `create-github-release.yml`, insert the same named step and ID shown in
  Step 1 immediately before `Create tag and GitHub release for a merged release
  PR`. Change that action's token input to:

  ```yaml
          token: ${{ steps.release-please-app-token.outputs.token }}
  ```

  Keep `target-branch`, `config-file`, `manifest-file`, and
  `skip-github-pull-request` unchanged.

- [ ] **Step 4: Check the workflow diff before committing**

  Run:

  ```bash
  git diff --check
  git diff -- .github/workflows/prepare-release.yml .github/workflows/create-github-release.yml
  ```

  Expected: only the token source changes plus one token-generation step per
  job; neither workflow exposes the private key or token value.

- [ ] **Step 5: Commit the workflow migration**

  ```bash
  git add .github/workflows/prepare-release.yml .github/workflows/create-github-release.yml
  git commit -m "ci: authenticate release please with GitHub App"
  ```

### Task 3: Add the copy-ready generic release runbook and align local docs

**Files:**

- Create: `docs/automated-npm-release.md`
- Modify: `RELEASING.md`
- Modify: `docs/superpowers/specs/2026-08-02-release-automation-design.md`
- Modify: `docs/superpowers/plans/2026-08-02-release-automation.md`

**Interfaces:**

- Consumes: the App credential names and workflow pattern defined in Task 2.
- Produces: a generic guide suitable for direct copying and repository-local
  references that no longer recommend a PAT.

- [ ] **Step 1: Write the generic runbook with portable sections**

  Create `docs/automated-npm-release.md` with these sections, in this order:

  ```markdown
  # Automated npm release runbook

  ## What this flow does
  ## Prerequisites
  ## Create the dedicated GitHub App
  ## Configure repository Actions credentials
  ## Configure the two Release Please workflows
  ## Normal release procedure
  ## Verify a release
  ## Troubleshooting and recovery
  ## Rotate the GitHub App private key
  ## Security rules
  ```

  Explain the lifecycle as conventional commits → manually dispatched Release
  Please PR → reviewed merge → GitHub Release and tag → npm publication with
  provenance. Use generic terms such as “the repository” and `vX.Y.Z` only.
  Include the exact token-generation snippet from Task 2, with the pinned
  action revision and credential names. Explain that each job creates a new
  one-hour token, the default post-step revokes it, and no cache, artifact,
  secret, or job output may store it.

- [ ] **Step 2: Include explicit recovery instructions**

  Under `## Troubleshooting and recovery`, include this decision table:

  | Symptom | Check | Recovery |
  | --- | --- | --- |
  | Token action cannot find an installation | App installation and selected repository | Install or reconfigure the App for only the intended repository, then rerun the workflow |
  | Token action reports insufficient permission | App and installation Contents, Pull requests, and Issues permissions | Grant the missing write permission, approve the installation update, then rerun the workflow |
  | Token action rejects credentials | client-ID variable and PEM secret | Correct the variable or replace the secret with a newly generated PEM key, then rerun the workflow |
  | Release Please fails after token creation | Release Please logs and repository state | Correct the release configuration or release-PR state, then rerun the relevant workflow; do not reuse the prior token |
  | A private key may be exposed | App key inventory and Actions secret history | Generate a new key, update the secret, validate a dispatch, delete the old key, and review workflow logs |

  State that retrying a workflow creates a new token and that recovery must not
  weaken selected-repository scope or enable token caching.

- [ ] **Step 3: Link the repository policy to the guide**

  Add this sentence immediately after the introductory paragraph in
  `RELEASING.md`:

  ```markdown
  For GitHub App setup, release operation, recovery, and credential rotation,
  see [Automated npm release runbook](docs/automated-npm-release.md).
  ```

  Keep the existing SemVer and Conventional Commit policy unchanged.

- [ ] **Step 4: Correct the superseded PAT documentation**

  In `docs/superpowers/specs/2026-08-02-release-automation-design.md`, replace
  the `Credentials and permissions` section so it specifies the dedicated
  per-repository GitHub App, `RELEASE_PLEASE_APP_ID`,
  `RELEASE_PLEASE_APP_PRIVATE_KEY`, fresh in-job token generation, explicit
  permission inputs, and the no-cache/no-reuse rule. Preserve the explanation
  that `GITHUB_TOKEN` alone cannot trigger the separate publication workflow.

  In `docs/superpowers/plans/2026-08-02-release-automation.md`, replace every
  `RELEASE_PLEASE_TOKEN` reference and its PAT setup instruction with the same
  GitHub App configuration and live-validation order. Do not add local terminal
  setup or tool-installation notes.

- [ ] **Step 5: Prove the guide is portable**

  Run:

  ```bash
  rg -n "pkg-nec|babel-plugin-istanbul|RELEASE_PLEASE_TOKEN|#12|8\.0\.3|https?://" docs/automated-npm-release.md
  ```

  Expected: no matches. Review every code block to confirm it uses generic
  paths, generic version placeholders, and the names defined by this plan.

- [ ] **Step 6: Commit the documentation**

  ```bash
  git add docs/automated-npm-release.md RELEASING.md docs/superpowers/specs/2026-08-02-release-automation-design.md docs/superpowers/plans/2026-08-02-release-automation.md
  git commit -m "docs: add automated npm release runbook"
  ```

### Task 4: Validate the migration and retire the PAT

**Files:**

- Verify: `.github/workflows/prepare-release.yml`
- Verify: `.github/workflows/create-github-release.yml`
- Verify: `.github/workflows/release-npm-package.yml`
- Verify: `docs/automated-npm-release.md`

**Interfaces:**

- Consumes: committed workflow changes, configured App credentials, and the
  unchanged `release.published` npm publication workflow.
- Produces: proof that the App executes the release lifecycle and authorization
  to remove the retired PAT secret.

- [ ] **Step 1: Validate static workflow and documentation changes**

  Run:

  ```bash
  actionlint .github/workflows/prepare-release.yml .github/workflows/create-github-release.yml .github/workflows/release-npm-package.yml
  git diff --check HEAD~2..HEAD
  git status --short
  ```

  Expected: `actionlint` and `git diff --check` exit successfully, and status
  shows no unintended changes. The npm-publication workflow remains unchanged.

- [ ] **Step 2: Run the repository test gate**

  Run:

  ```bash
  yarn test
  ```

  Expected: StandardJS linting, the Babel build, and the NYC/Mocha test suite
  succeed. If the build changes generated `lib/` files, do not add them unless
  they were intentionally changed by this migration.

- [ ] **Step 3: Exercise release-PR creation with the App**

  Manually dispatch `Prepare release` on `main`. Confirm that it creates or
  refreshes the pending Release Please PR, and that the GitHub activity is
  attributed to the dedicated App. Inspect the workflow log only for action
  success and permission errors; do not copy any token value into records.

- [ ] **Step 4: Exercise the release and publication chain**

  After reviewing and merging the Release Please PR, confirm that `Create
  GitHub release` creates exactly one `vX.Y.Z` tag and published GitHub
  Release. Confirm that the existing release-published workflow starts,
  verifies tag and package version, runs its tests, and publishes with npm
  provenance.

- [ ] **Step 5: Remove the retired PAT only after success**

  In repository Actions secrets, delete `RELEASE_PLEASE_TOKEN`. Trigger or
  inspect one additional harmless Release Please no-op run to confirm the
  workflows depend solely on the App variable and private-key secret. Record
  the successful release URL in the release discussion or release notes, not a
  source-file commit.
