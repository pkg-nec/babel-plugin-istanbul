# Release Versioning Guide

This guide is the canonical policy for maintainers selecting release-affecting
Conventional Commit subjects and deciding this package's SemVer version.

## Release workflow

The release process is split across three workflows:

- `prepare-release.yml` prepares the reviewed release PR.
- `create-github-release.yml` creates the tag and GitHub Release after that PR
  is merged.
- `release-npm-package.yml` tests and publishes the released tag with
  provenance.

The release commit subject is the subject that lands on `main`. For a squash
merge, this is normally the PR title; for a merge commit, it is the
merge-commit subject.

Release Please recognizes `feat`, `fix`, and `deps` as releasable types. A
`docs` or `ci` commit by itself does not produce a release proposal for this
Node package.

## Selecting a prefix

| Situation | Commit prefix |
| --- | --- |
| Ordinary compatible dependency upgrade | `deps:` |
| Dependency upgrade that resolves a user-impacting defect or CVE | `fix(deps):` |
| New user-facing package capability | `feat:` |
| User-visible incompatibility | `deps!:` with a `BREAKING CHANGE:` footer |

A dependency's major version alone does not make this package's release major.
Before deciding, test the supported Babel 7/Babel 8 and Node matrix. Declare a
major release only when an upgrade changes the public contract, such as raising
the supported Node floor or changing documented instrumentation, include, or
exclude behavior.

## Merge policy

Normal merge commits are supported, but squash merges are preferred so `main`
has one clean release entry. When normal merging, use non-releasable prefixes
for intermediate commits that must not appear as individual changelog entries.

Do not trigger a release solely to publish `docs:` or `ci:` work.
