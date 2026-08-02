# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Shared CI for the H8IO Scala projects: reusable workflows under `.github/workflows/` and composite
actions under `actions/`. Nothing here builds or runs on its own — every entry point exists to be
called from another repository, e.g. `h8io/sbt-scoverage-summary` (a sibling checkout at `../sbt-scoverage-summary`),
whose workflows are three-line stubs delegating to `h8io/gha/.github/workflows/test.yaml@v4`.

## Commands

There is no build. `linter.yaml` is the entire test suite, and it is worth running locally before pushing:

```bash
curl -sSfL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash \
  | bash -s -- latest ./bin
./bin/actionlint -color -shellcheck=shellcheck        # workflows only
npx --yes @action-validator/cli ./actions/<name>/action.yaml   # schema only, per action
```

`actionlint` does not descend into `actions/*/action.yaml`, and `@action-validator/cli` validates schema
rather than shell. So **the shell inside a composite action is checked by neither**. When editing one, pull
its `run:` bodies out (substituting the `${{ … }}` expressions with placeholders) and run `shellcheck` on
them by hand.

## Versioning is the thing to understand first

A single tag series covers the whole repository. `floating-tags.yaml` fires on any `vX.Y.Z` push and
force-moves `vX` and `vX.Y` onto it, so consumers pinning `@v4` follow the newest `v4.*` automatically.

Two consequences that are easy to get wrong:

**A merge to `main` changes nothing.** Consumers resolve `@v4`, so an edit only reaches them once a new
patch tag is pushed and the floating tag advances. "Merged" and "released" are different events here.

**Workflows pin the actions of this same repository by floating tag too** — `uses: h8io/gha/actions/…@v3`
while the repository sits at `v4`. That is not a mistake in isolation, but it means a workflow and the
action it calls can come from different releases: a change to an action is invisible to a workflow still
pinned at an older series. When changing an action that a workflow uses, bump the `uses:` reference in
that workflow to the series being released, or the fix ships dead.

Everything moves together under one tag, so workflows and actions cannot be versioned independently.

## Contracts with the calling repository

- `test.yaml` runs `./test.sh` from the caller's root — the caller owns the whole build, this repo only
  provides the runner, the Coursier cache and the JVM. It then invokes `publish-scoverage-summary`, which
  expects `sbt-scoverage-summary` to have produced `**/target/**/scoverage-summary/gfm.md`.
- `release.yaml` refuses to release unless the pushed tag points at the exact commit of the default
  branch, then runs `sbt "cleanFull; +test; ci-release"`. It needs the PGP and Sonatype secrets, which are
  passed through `secrets: inherit` from the caller.
- `scala-steward.yaml` authenticates as a GitHub App (`H8IO_APP_ID` / `H8IO_APP_PK`), not as a user.

Each workflow declares its own `permissions`; `test.yaml` needs `pull-requests: write` purely to post the
coverage comment.

## Action-specific notes

`check-sonatype-token` treats only `401` as invalid credentials and accepts `200|400|403|404`. The
endpoint is queried with a deliberately bogus id, so anything other than an authentication failure means
the credentials were good enough to get that far.

`floating-tags` checks itself out with `fetch-depth: 0` and force-pushes, since it rewrites existing tags.
