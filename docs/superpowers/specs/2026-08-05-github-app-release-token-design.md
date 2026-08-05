# GitHub App release-token design

## Goal

Replace the long-lived Release Please personal access token with a short-lived
GitHub App installation token. Each repository in the organization owns a
dedicated GitHub App that is installed only on that repository. Preserve the
reviewed Release Please PR, GitHub Release, and npm provenance-publication
flow while documenting a reusable operating model for other npm packages.

## Repository App model

Create one organization-owned GitHub App for each repository that adopts this
release process. Each App has its own client ID and private key, and is
installed on its corresponding repository only. The App needs these repository
permissions:

- Contents: read and write
- Pull requests: read and write
- Issues: read and write

The App has no webhook-event requirement because GitHub Actions invokes it
directly. Its installation must use selected-repository access and select only
the repository that owns the App's release workflow.

The repository stores the App client ID in the Actions variable
`RELEASE_PLEASE_APP_ID` and the PEM private key in the Actions secret
`RELEASE_PLEASE_APP_PRIVATE_KEY`. The existing
`RELEASE_PLEASE_TOKEN` secret is removed only after the migrated workflows
have passed live validation.

## Workflow token flow

Both `prepare-release.yml` and `create-github-release.yml` create a token in
the same job that runs Release Please:

1. `actions/create-github-app-token` authenticates with the repository App's
   client ID and private key.
2. The action explicitly requests Contents, Pull requests, and Issues write
   permissions and, with no cross-repository targeting inputs, scopes the
   installation token to the current repository.
3. `googleapis/release-please-action` receives the generated step output as
   its `token` input.
4. The token action's default post-step revokes the token when the job ends.

The workflow must pin the token action to a reviewed immutable revision in the
currently supported v3 release line, following the repository's existing
action-pinning convention. The existing Release Please action revision and
release flow remain otherwise unchanged.

The generated token is never saved as an Actions secret, cache entry, artifact,
or job output. GitHub App installation tokens expire after one hour and the
token action revokes them at normal job completion. The two release workflows
are independent runs, so each run creates its own short-lived token. This is
the intended reuse boundary; token caching is neither necessary nor permitted.

## Release behavior and failure handling

The migration does not change the release lifecycle:

1. A maintainer manually dispatches `Prepare release` to create or update a
   Release Please PR.
2. A maintainer reviews and merges that PR.
3. The `main` push runs `Create GitHub release`, which creates the tag and
   published GitHub Release for a merged pending Release Please PR.
4. The GitHub Release event starts the existing npm provenance publication
   workflow.

The first operational test must confirm that the App creates or updates the
release PR and that a merged release PR starts the unchanged publication chain.
If authentication fails, maintainers verify the App installation, selected
repository scope, installation approval, client-ID variable, private-key
secret, and the three required permissions. Rotate a compromised or expiring
private key by generating a replacement in the App settings, updating the
repository secret, validating a dispatch, then deleting the retired key.

## Generic release guide

Add `docs/automated-npm-release.md` as a self-contained guide that can be
copied to another npm-package repository without edits. It must not name this
organization, repository, package, historical pull request, release version,
or repository-specific URL.

The guide describes the generic architecture and normal lifecycle, conventional
commit expectations, dedicated per-repository GitHub App setup, selected-repo
installation, minimum permissions, Actions variable/secret names, workflow
token-generation pattern, and the no-cache rule. It also includes operational
procedures for manual dispatch, release verification, token/authentication
diagnosis, App-private-key rotation, retrying failed automation, and safe
release recovery. Reusable snippets use only standard action names, generic
workflow paths, and generic version placeholders.

`RELEASING.md` remains the repository-specific policy document and links to
the generic guide. The prior release-automation design and implementation plan
are updated to replace their PAT instructions with this GitHub App model.

## Validation

Validate both changed workflow files with `actionlint`, check the patch with
`git diff --check`, and confirm the generic guide has no repository-specific
references. Run the repository test gate because the GitHub Release to npm
publication path remains part of the protected release contract. Finally,
exercise a manual release preparation and a reviewed release-PR merge to prove
the App-authenticated end-to-end flow before deleting the old PAT.
