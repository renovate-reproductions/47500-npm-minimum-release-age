# Reproduction for the minimumReleaseAge package-manager stack

Shows how Renovate passes the repository/manager `minimumReleaseAge` to npm, pnpm, Poetry and cargo, reads their own configured ages, and when lock file maintenance (LFM) skips the stability wait.

Pull requests of renovatebot/renovate (all drafts, bottom to top, GitHub Stack 47679):

1. [#47570](https://github.com/renovatebot/renovate/pull/47570): core, the repository value is passed instead of the packageRules value; extracted package-manager ages become packageRules
2. [#47677](https://github.com/renovatebot/renovate/pull/47677): Poetry
3. [#47678](https://github.com/renovatebot/renovate/pull/47678): `minimumReleaseAgePackageManager` and the `security:minimumReleaseAge*` presets
4. [#47500](https://github.com/renovatebot/renovate/pull/47500): npm
5. [#47542](https://github.com/renovatebot/renovate/pull/47542): pnpm
6. [#47543](https://github.com/renovatebot/renovate/pull/47543): `rust-toolchain` constraint
7. [#47544](https://github.com/renovatebot/renovate/pull/47544): cargo

## Config

- npm and cargo get `minimumReleaseAge: 7 days` at manager level. This is the repository value.
- Poetry gets no `minimumReleaseAge`. It relies on `security:minimumReleaseAgePyPI` (3 days in `packageRules` and `poetry.minimumReleaseAgePackageManager: 3 days`).
- Every directory is a separate project with a committed lock file that is older than today, so LFM has updates to make. LFM creates one branch per directory (`additionalBranchPrefix: "{{packageFileDir}}-"`).
- Renovate reads only the `.cargo/config.toml` in the repository root. Each cargo config variant is therefore its own base branch (`cargo-global-age`, `cargo-allow`, `cargo-registry-age`) that adds only that file to `main`.
- Only the packages of the cases get update PRs (`semver`, `@types/node`, `express`, `requests`, `boto3`, `serde`, `cc`).

## npm

On `main` the `.npmrc` ages are ignored and LFM always waits.

| Directory             | Setup                                                 | PR             | Expected                                                                                                                                                      |
| --------------------- | ----------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `npm/before`          | `.npmrc` `before=2024-06-01`                          | #47500         | Lookup proposes `semver` only up to 7.6.2. Commands get `--before=2024-06-01…` (the older of the two dates). LFM `renovate/stability-days` passes right away. |
| `npm/min-release-age` | `min-release-age=7`, `packageManager: npm@11.10.0`    | #47500         | No `--before` (npm rejects both). LFM passes right away.                                                                                                      |
| `npm/no-constraint`   | `min-release-age=7`, no npm version constraint        | #47500         | LFM stays pending.                                                                                                                                            |
| `npm/yarn`            | `yarn.lock`, `.npmrc` ignored                         | #47500         | LFM stays pending.                                                                                                                                            |
| `npm/repository-age`  | no `.npmrc`, `express` 4 to 5 with 30 days for majors | #47570, #47500 | The major branch passes `--before` = now minus 7 days (main: now minus 30 days).                                                                              |
| `npm/exemption`       | `@types/node` with `minimumReleaseAge: null`          | #47500         | If the newest `@types/node` is younger than 7 days: no `--before` for that branch, plus a PR notice.                                                          |

## pnpm

pnpm 11.28.8.

| Directory           | Setup                                                                 | PR     | Expected                                                                                                                                                                                                                                                                                                                |
| ------------------- | --------------------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm/default`      | no pnpm age, `packageManager: pnpm@11.28.8`                           | #47542 | Renovate uses pnpm's v11 default of 1 day for npm packages (extracted package rule), so `@types/node` with `minimumReleaseAge: null` still waits 1 day. Branches pass `--config.minimum-release-age=10080` and `--config.minimum-release-age-exclude=<name>@<version>` for each updated package (a single entry twice). |
| `pnpm/workspace`    | `minimumReleaseAge: 14400`, `minimumReleaseAgeExclude: [@types/node]` | #47542 | Lookup waits 10 days except for `@types/node` (7 days). The flag is `--config.minimum-release-age=14400`; the exclude flags repeat `@types/node` first. LFM passes right away.                                                                                                                                          |
| `pnpm/strict-false` | as above plus `minimumReleaseAgeStrict: false`                        | #47542 | LFM stays pending.                                                                                                                                                                                                                                                                                                      |

## Poetry

Poetry 2.4 or newer (locked with 2.5.1).

| Directory            | Setup                                                        | PR             | Expected                                                                                                                                                                                                              |
| -------------------- | ------------------------------------------------------------ | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `poetry/no-config`   | no `poetry.toml`                                             | #47677, #47678 | `POETRY_SOLVER_MIN_RELEASE_AGE=3` from the preset's `minimumReleaseAgePackageManager` (#47678). With only #47677 no variable is set, as the 3 days are only in `packageRules` (main: 3, from the update's own value). |
| `poetry/poetry-toml` | `min-release-age = 5`, `min-release-age-exclude = ["boto3"]` | #47677         | Lookup waits 5 days for `requests` and 3 days for `boto3`. No variable is set, as max(3, 5) equals Poetry's own value.                                                                                                |
| `poetry/exemption`   | `boto3` with `minimumReleaseAge: null`                       | #47677         | If the newest `boto3` is younger than 3 days: `POETRY_SOLVER_MIN_RELEASE_AGE=0` for that branch, plus a PR notice.                                                                                                    |

## cargo

Set `RENOVATE_X_CARGO_MIN_PUBLISH_AGE=true` for the LFM cases.

| Directory (or base branch)    | Setup                                                           | PR             | Expected                                                                                                                                                                                                          |
| ----------------------------- | --------------------------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `main`: `cargo/toolchain`     | `rust-toolchain` `1.100.0` next to `rust-toolchain.toml` `1.99` | #47543, #47544 | `rust` constraint is 1.100.0 (rustup prefers `rust-toolchain`). `CARGO_REGISTRY_GLOBAL_MIN_PUBLISH_AGE` is 7 days (largest whole unit) and `CARGO_RESOLVER_INCOMPATIBLE_PUBLISH_AGE=deny`. LFM passes right away. |
| `main`: `cargo/old-toolchain` | uses `cargo/rust-toolchain.toml` `1.99`                         | #47543, #47544 | LFM stays pending (cargo older than 1.100 ignores the age).                                                                                                                                                       |
| `main`: `cargo/exemption`     | `cc` with `minimumReleaseAge: null`                             | #47544         | If the newest release is younger than 7 days: neither variable is set for that branch, plus a PR notice.                                                                                                          |
| `cargo-global-age`            | `registry.global-min-publish-age = "10 days"`                   | #47544         | Lookup waits 10 days. No variable is set, as cargo's own 10 days are larger. LFM of `cargo/toolchain` passes right away.                                                                                          |
| `cargo-allow`                 | `resolver.incompatible-publish-age = "allow"`                   | #47544         | Updates still pass `deny`. LFM of `cargo/toolchain` stays pending (cargo's `allow` applies).                                                                                                                      |
| `cargo-registry-age`          | `registries.crates-io.min-publish-age = "2 days"`               | #47544         | Lookup still waits 7 days. LFM of `cargo/toolchain` stays pending (the debug log names the registry). `cargo/exemption` is not lowered, as cargo applies the registry's own age.                                  |

## Verify

Check out the top branch `feat/cargo-minimum-release-age-extracted` of renovatebot/renovate (or the PR branch of interest) and run `pnpm install`. Poetry 2.4 or newer, pnpm 11.28.8 and cargo must be on `PATH`. Then run:

```sh
LOG_LEVEL=debug RENOVATE_TOKEN=<token> RENOVATE_X_CARGO_MIN_PUBLISH_AGE=true \
  node lib/renovate.ts --binary-source=global --dry-run=full \
  renovate-reproductions/47500-npm-minimum-release-age
```

Search the log for:

- `--before`
- `minimum-release-age`
- `POETRY_SOLVER_MIN_RELEASE_AGE`
- `CARGO_REGISTRY_GLOBAL_MIN_PUBLISH_AGE`
- `the package manager enforces the minimum release age`
- `minimumReleaseAgeExtracted`
