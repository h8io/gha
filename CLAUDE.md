# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Shared CI for the H8IO Scala projects: reusable workflows under `.github/workflows/` and composite
actions under `actions/`. Nothing here builds or runs on its own — every entry point exists to be
called from another repository, e.g. `h8io/sbt-scoverage-summary` (a sibling checkout at `../sbt-scoverage-summary`),
whose workflows are three-line stubs delegating to `h8io/gha/.github/workflows/test.yaml@v5`.

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

Every entry point that builds anything does it by running a script from the caller's root, named after the
workflow that runs it: `./test.sh`, `./release.sh`, `./snapshot.sh`. The caller owns the build and the build
tool; this repo provides the runner, the Coursier cache, the JVM, the secrets and the policy around them. No
sbt command is written down here, and none should be — that is what pinned `stages` to the `v3` series for
as long as `release.yaml` spelled out `cleanFull`.

- `test.yaml` runs `./test.sh`, then invokes `publish-scoverage-summary`, which expects
  `sbt-scoverage-summary` to have produced `**/target/**/scoverage-summary/gfm.md`.
- `release.yaml` refuses to release unless the pushed tag points at the exact commit of the default
  branch, then runs `./release.sh`. With `publish-pages` on, that script is also expected to leave the
  documentation site in `target/pages`. `snapshot.yaml` runs `./snapshot.sh`. Both need the PGP and
  Sonatype secrets, which are passed through `secrets: inherit` from the caller.
- `scala-steward.yaml` authenticates as a GitHub App (`H8IO_APP_ID` / `H8IO_APP_PK`), not as a user.

Each workflow declares its own `permissions`; `test.yaml` needs `pull-requests: write` purely to post the
coverage comment.

### Documentation is deployed by the release, not next to it

A tag push used to start the release and the site deployment as two unrelated workflows, so a release that
failed — wrong tag, expired Sonatype token, red `+test` — still left the site describing a version that
never reached Maven Central, and with the site unversioned there was nothing to fall back to.

The site is therefore built by the same `./release.sh`, in the same sbt run, that publishes the artifact. A
separate workflow would mean a second checkout, a second JVM and a second full build, producing
documentation for a commit the first build has already compiled. What is left for this repo is to upload
`target/pages` and deploy it from a job that `needs: release`. The upload is an ordinary step carrying the
implicit `success()`, and that is the entire gate: a release that failed uploads nothing, so the deployment
has nothing to deploy.

`publish-pages` is off by default — a project whose documentation is a `README.md` in the root has nothing
to deploy. The `pages: write` and `id-token: write` the deployment needs are declared on that job alone and
never at the workflow level, so that callers leaving the flag off do not have to grant them. For the same
reason there is no `configure-pages` step: it would have to run in the release job, where it would need
`pages: read` in order to produce outputs that a site built by sbt does not read anyway.

Deploying documentation without releasing is deliberately not possible, since an unversioned site describes
whatever the latest release is. When the release succeeded and only the deployment failed, re-run the
`pages` job of that run — the uploaded site is retained for a week so that this keeps working.

The reverse gap stays open by design: `gh release create` runs before the deployment, so a failed deploy
leaves a released artifact next to an older site. Documentation lagging behind the artifact is the harmless
direction; documentation running ahead of it is not.

## Action-specific notes

`check-sonatype-token` treats only `401` as invalid credentials and accepts `200|400|403|404`. The
endpoint is queried with a deliberately bogus id, so anything other than an authentication failure means
the credentials were good enough to get that far.

`floating-tags` checks itself out with `fetch-depth: 0` and force-pushes, since it rewrites existing tags.
