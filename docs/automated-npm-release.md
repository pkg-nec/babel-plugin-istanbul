# Automated npm release runbook

## What this flow does

Conventional commits are collected into a manually dispatched Release Please
pull request. After maintainers review and merge that pull request, Release
Please creates a GitHub Release and the matching `vX.Y.Z` tag. The published
release then starts npm publication with provenance.

## Prerequisites

The repository must have Release Please configuration, a workflow that prepares
release pull requests, and a separate workflow that creates releases after a
release pull request is merged. The npm publication workflow must run on the
published-release event and must retain the permissions required for npm
provenance.

## Create the dedicated GitHub App

Create one organization-owned GitHub App for the repository's Release Please
flow. Leave webhook events inactive, restrict its installation to the selected
repository, and grant it write access to Contents, Pull requests, and Issues.
Generate a private key for the App and retain its client ID for the repository
configuration.

## Configure repository Actions credentials

Create the repository Actions variable `RELEASE_PLEASE_APP_ID` with the App
client ID. Create the repository Actions secret `RELEASE_PLEASE_APP_PRIVATE_KEY`
with the App private key PEM. Do not expose the PEM in workflow logs.

Each Release Please job creates a fresh one-hour installation token. The action's
default post-step revokes that token. Do not cache, store in an artifact or
secret, pass through a job output, or otherwise reuse the token.

## Configure the two Release Please workflows

Add this token-generation step to each Release Please job, before the Release
Please action:

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

Configure the Release Please action in each job to use
`${{ steps.release-please-app-token.outputs.token }}`. The manually dispatched
workflow creates or refreshes a release pull request. The workflow that runs
after a merge creates a GitHub Release only for a merged pending release pull
request. Explicit permission inputs keep the generated token limited to the
access Release Please needs.

## Normal release procedure

1. Merge releasable work using Conventional Commit subjects.
2. Manually dispatch the workflow that prepares the Release Please pull request.
3. Review the generated version and changelog changes, then merge the release
   pull request.
4. Confirm that the release workflow creates the matching `vX.Y.Z` tag and
   GitHub Release.
5. Confirm that the published release starts npm publication with provenance.

## Verify a release

Confirm the release pull request reflects the expected Conventional Commit
changes. After it is merged, confirm one GitHub Release and its matching
`vX.Y.Z` tag were created. Confirm the npm publication workflow ran from the
published-release event, verified the tag and package version, completed its
tests, and published with provenance.

## Troubleshooting and recovery

| Symptom | Check | Recovery |
| --- | --- | --- |
| Token action cannot find an installation | App installation and selected repository | Install or reconfigure the App for only the intended repository, then rerun the workflow |
| Token action reports insufficient permission | App and installation Contents, Pull requests, and Issues permissions | Grant the missing write permission, approve the installation update, then rerun the workflow |
| Token action rejects credentials | client-ID variable and PEM secret | Correct the variable or replace the secret with a newly generated PEM key, then rerun the workflow |
| Release Please fails after token creation | Release Please logs and repository state | Correct the release configuration or release-PR state, then rerun the relevant workflow; do not reuse the prior token |
| A private key may be exposed | App key inventory and Actions secret history | Generate a new key, update the secret, validate a dispatch, delete the old key, and review workflow logs |

Retrying a workflow creates a new token. Recovery must not weaken
selected-repository scope or enable token caching.

## Rotate the GitHub App private key

Generate a new private key for the dedicated App, replace
`RELEASE_PLEASE_APP_PRIVATE_KEY`, and validate a manually dispatched workflow.
After validation, delete the old App key. Review the workflow logs and key
inventory if the rotation followed a suspected exposure.

## Security rules

Install the App only on the intended repository. Keep Contents, Pull requests,
and Issues permissions limited to what Release Please requires. Use a fresh
one-hour token in each Release Please job and rely on the default post-step to
revoke it. Never cache or reuse a token, place it in an artifact, secret, or
job output, or expose it in logs.
