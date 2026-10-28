# Reproduction for renovatebot/renovate#47500 and #47501

Shows how the npm manager reads the minimum release age from `.npmrc` and when lock file maintenance (LFM) skips the stability wait because npm enforces the age.

- [#47501](https://github.com/renovatebot/renovate/pull/47501): managers report `minimumReleaseAgeExtracted` and `enforcesMinimumReleaseAge()`; LFM skips the stability wait when the manager enforces the age
- [#47500](https://github.com/renovatebot/renovate/pull/47500): npm reads `before` / `min-release-age` from the repository `.npmrc`; effective age is `max(minimumReleaseAge, extracted)`

## Layout

All directories are separate projects with a committed lock file, generated with an older `--before` date so that LFM has updates to make.
Renovate is configured with `minimumReleaseAge: "3 days"` and `minimumReleaseAgeBehaviour: "timestamp-required"`.
LFM is normally one branch for all package files, so `additionalBranchPrefix: "{{packageFileDir}}-"` gives each directory its own branch.

| Directory | `.npmrc` | Package manager constraint | Enforces age? | Expected |
| --- | --- | --- | --- | --- |
| `before/` | `before=2024-06-01` | none | yes (`before`) | `semver` 7.5.0 is only proposed up to 7.6.2 (newest release before the date); LFM branch and PR are created right away |
| `min-release-age/` | `min-release-age=7` | `packageManager: npm@11.10.0` | yes (npm >= 11.10.0 is guaranteed) | LFM branch and PR are created right away |
| `no-constraint/` | `min-release-age=7` | none | no (`--before` is not passed next to `min-release-age`, and npm >= 11.10.0 is not guaranteed) | LFM is held back by the stability wait, pending on the Dependency Dashboard |
| `yarn/` | `min-release-age=7` (ignored) | `packageManager: yarn@1.22.22` | no | LFM is held back by the stability wait, pending on the Dependency Dashboard |

## Expected behaviour

- On `main`: LFM always waits for the release timestamps, and the `.npmrc` ages are ignored when looking up updates.
- On the PR branch: the LFM branches for `before/` and `min-release-age/` are created immediately (debug log: "the package manager enforces the minimum release age"), `no-constraint/` and `yarn/` wait.

## Verify

Check out the PR branch `feat/npm-minimum-release-age-extracted` of renovatebot/renovate (npm >= 11.10.0 must be available), then run:

```sh
LOG_LEVEL=debug RENOVATE_TOKEN=<token> \
  node lib/renovate.ts renovate-reproductions/47500-npm-minimum-release-age
```

In the debug log, look for "the package manager enforces the minimum release age" and the `--before` flag of the npm commands.
