# Release automation design

## Goal

Automate version calculation, changelog generation, GitHub releases, and npm
publication for `@pkg-nec/babel-plugin-istanbul` while preserving a maintainer
review gate before every release.

## Release model

The repository has three workflows with distinct responsibilities:

1. `prepare-release.yml` is manually dispatched when maintainers decide that
   merged work is ready to release. It creates or refreshes a Release Please
   pull request; it does not tag a release or publish a package.
2. `create-github-release.yml` runs after every push to `main`. It does not
   create release pull requests. It creates a tag and a published GitHub
   release only when it finds a merged, pending Release Please pull request.
3. `release-npm-package.yml` runs when a GitHub release is published. It checks
   out the released tag, verifies that the tag matches `package.json`, tests
   the package, and publishes it to npm with provenance.

The existing `release.yml` workflow will be renamed to
`release-npm-package.yml`; its behavior remains unchanged.

## Configuration and bootstrap

Add a manifest configuration at the repository root:

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

Store the established release in `.release-please-manifest.json`:

```json
{
  ".": "8.0.3"
}
```

The bootstrap SHA is the `v8.0.3` commit. It bounds the initial collection of
commits. Once a Release Please release PR is merged, release history and the
manifest become the source of truth for subsequent releases.

## Prepare-release workflow

`prepare-release.yml` is triggered by `workflow_dispatch`, targets `main`, and
uses `googleapis/release-please-action@v5.0.0` with the manifest config and:

```yaml
skip-github-release: true
```

It creates or updates a single Release Please PR that changes `package.json`,
`CHANGELOG.md`, and `.release-please-manifest.json`. Maintainers review and
merge this PR. If normal work merges while the release PR is open, a maintainer
dispatches this workflow again before merging the release PR.

## Create-GitHub-release workflow

`create-github-release.yml` runs on every push to `main` and uses the same
configuration and action version with:

```yaml
skip-github-pull-request: true
```

It does not release based on the triggering commit message. Instead, Release
Please queries GitHub for a merged Release Please PR in its pending lifecycle.
An ordinary merged `fix(deps)` PR therefore causes a no-op in this workflow.
After a reviewed Release Please PR is merged, this workflow creates the
matching `vX.Y.Z` tag and a published GitHub Release. The latter triggers
`release-npm-package.yml`.

## Versioning convention

Release Please derives the next version from Conventional Commit messages:

- `fix(deps): ...` produces a patch release.
- `feat: ...` produces a minor release.
- A `!` or `BREAKING CHANGE:` footer produces a major release.

Dependency security fixes must be merged with `fix(deps): ...` subjects rather
than `chore(deps): ...`, so they are included in a release proposal.

## Credentials and permissions

Both Release Please workflows use a dedicated secret named
`RELEASE_PLEASE_TOKEN`, preferably a fine-grained PAT owned by a release bot or
service account and limited to this repository. It must grant the action the
repository contents, pull-request, and issue permissions needed to create and
label release PRs and create tags/releases.

The token must not be only `GITHUB_TOKEN`: a GitHub release created with that
token would not trigger the separate npm-publishing workflow. A PAT or GitHub
App token permits the `release.published` event to trigger
`release-npm-package.yml`.

## Validation

Before enabling the workflow on `main`, validate it in a disposable branch or
test repository with a `fix(deps): ...` commit. Confirm that the manual workflow
opens a patch release PR, an ordinary code merge does not create a GitHub
release, and merging the release PR creates `vX.Y.Z`, starts the npm workflow,
and publishes only after tests pass.
