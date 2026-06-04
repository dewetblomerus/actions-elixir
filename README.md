# actions-elixir

Shared GitHub Actions for Elixir projects that use `mix check` as the required CI gate and `mix_audit` as a separate informational check.

Each application owns its own `mix.exs` and decides what `mix check` means. This repository only handles GitHub Actions boilerplate: checkout, BEAM setup, Mix dependency and build caches, dependency installation, running the required check command, and running dependency audit separately.

## Reusable workflow

Use this when the default job shape works for the app. It includes a Postgres service on `localhost:5432` with username `postgres` and password `postgres`, matching the Phoenix-style test setup used by `red`.

Create `.github/workflows/elixir.yml` in the application repository:

```yaml
name: Checks

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  elixir-ci:
    name: Elixir CI
    uses: dewetblomerus/actions-elixir/.github/workflows/elixir-ci.yml@main
```

If the app has a `.tool-versions` file, `erlef/setup-beam` reads Erlang and Elixir versions from it. If not, pass exact versions:

```yaml
jobs:
  elixir-ci:
    name: Elixir CI
    uses: dewetblomerus/actions-elixir/.github/workflows/elixir-ci.yml@main
    with:
      otp-version: "29.1"
      elixir-version: "1.19.5-otp-29"
```

For branch protection with the reusable workflow, require the `Elixir CI / Mix Check` check. The reusable workflow also creates an `Elixir CI / Mix Audit` check, but it is allowed to fail by default so dependency advisories show up without blocking PR merges.

Examples for the existing apps:

```yaml
# Use checked-in .tool-versions
jobs:
  elixir-ci:
    name: Elixir CI
    uses: dewetblomerus/actions-elixir/.github/workflows/elixir-ci.yml@main
```

## Composite action

Use this when the application needs to define its own services, permissions, matrix, or extra steps. The repo workflow owns the job and calls the shared action inside the steps.

```yaml
name: Checks

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  MIX_ENV: test

permissions:
  contents: read

jobs:
  mix-check:
    name: Mix Check
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-retries 5
          --health-timeout 5s
    steps:
      - uses: actions/checkout@v6
      - uses: dewetblomerus/actions-elixir/.github/actions/mix-check@main

  mix-audit:
    name: Mix Audit
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - uses: actions/checkout@v6
      - uses: dewetblomerus/actions-elixir/.github/actions/mix-audit@main
```

For branch protection with the composite action, require only the `Mix Check` job.

## Elixir version bump action

Use this when an application should get a pull request that bumps it to the shared Elixir v1 tuple used by `quick-average`: Elixir `1.19.5`, Erlang `28.5`, Docker OTP `28.5.0.1`, and Debian `trixie-20260518-slim`.

Create a manually triggered workflow in the application repository:

```yaml
name: Bump Elixir

on:
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  bump-elixir:
    name: Bump Elixir
    runs-on: ubuntu-latest
    steps:
      - uses: dewetblomerus/actions-elixir/.github/actions/bump-elixir@main
        with:
          token: ${{ secrets.CREATE_PULL_REQUEST_TOKEN }}
```

The bump action checks out the repository itself, so the workflow does not need a separate `actions/checkout` step.

The `token` input must be a token that can create the bump branch and pull request in the target repository. In the existing apps this is stored as `CREATE_PULL_REQUEST_TOKEN`.

The action updates root-level `.tool-versions`, `Dockerfile`, and `mix.exs` when those files exist, then opens a pull request assigned to `dewetblomerus`. It intentionally leaves Dockerfile comments unchanged.

The bump action supports these inputs:

| Input | Default | Purpose |
| --- | --- | --- |
| `assignees` | `dewetblomerus` | GitHub usernames assigned to the pull request. |
| `branch` | `bump-elixir-1.19` | Branch name used for the version bump pull request. |
| `commit-message` | `Bump Elixir to 1.19.5` | Commit message for the version bump. |
| `pr-body` | Version tuple summary | Pull request body. |
| `pr-title` | `Bump Elixir to 1.19.5` | Pull request title. |
| `token` | Required | GitHub token used to create the pull request. Use a token that can create branches and pull requests in the target repository. |

## Inputs

Both mix check entry points support these inputs:

| Input | Default | Purpose |
| --- | --- | --- |
| `cache-version` | `v1` | Change this to force fresh Mix caches. |
| `check-command` | `mix check --except mix_audit` | Command that must pass before merge. |
| `deps-command` | `mix deps.get` | Dependency installation command. |
| `elixir-version` | `1.19.5-otp-29` | Used only when `version-file` does not exist. |
| `mix-env` | `test` | `MIX_ENV` for dependency installation and checks. |
| `otp-version` | `29.1` | Used only when `version-file` does not exist. |
| `version-file` | `.tool-versions` | Version file read by `erlef/setup-beam`. |
| `working-directory` | `.` | Directory containing `mix.exs`. |

The reusable workflow also supports `runs-on`, defaulting to `ubuntu-latest`.

The reusable workflow and `mix-audit` composite action also support:

| Input | Default | Purpose |
| --- | --- | --- |
| `audit-command` | `mix deps.audit` | Command that runs the informational dependency audit. |

The reusable workflow also supports:

| Input | Default | Purpose |
| --- | --- | --- |
| `audit-continue-on-error` | `true` | Allows `Mix Audit` to fail without failing the workflow. |

## Configuring `mix check`

Add `ex_check` to each application and configure aliases locally. A typical Phoenix app can start with:

```elixir
def project do
  [
    # ...
    aliases: aliases()
  ]
end

def cli do
  [
    preferred_envs: [check: :test]
  ]
end

defp deps do
  [
    {:ex_check, "~> 0.16.0", only: [:dev, :test], runtime: false},
    {:mix_audit, "~> 2.1", only: [:dev, :test], runtime: false}
  ]
end

defp aliases do
  [
    check: [
      "compile --warnings-as-errors",
      "deps.unlock --check-unused",
      "format --check-formatted",
      "test",
      "credo --strict",
      "sobelow --config",
      "deps.audit"
    ]
  ]
end
```

Tune the list per repo. For example, apps without Sobelow should remove that task, and apps with asset or Dialyzer checks can add them. The shared action runs `mix check --except mix_audit` by default, so the `mix_audit` check is excluded from the required job and run separately as `Mix Audit`.

## Notes

This setup follows the useful parts of Felt's Ultimate Elixir CI pattern: shared setup, dependency/build caching, a single required quality gate, and a separate informational dependency audit. It intentionally leaves the actual quality policy in each app's `mix.exs` so every repository can evolve its checks independently while GitHub Actions stays shared.
