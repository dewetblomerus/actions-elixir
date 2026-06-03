# actions-elixir

Shared GitHub Actions for Elixir projects that use `mix check` as the single CI gate.

Each application owns its own `mix.exs` and decides what `mix check` means. This repository only handles GitHub Actions boilerplate: checkout, BEAM setup, Mix dependency and build caches, dependency installation, and running the check command.

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
  mix-check:
    uses: dewetblomerus/actions-elixir/.github/workflows/mix-check.yml@main
```

If the app has a `.tool-versions` file, `erlef/setup-beam` reads Erlang and Elixir versions from it. If not, pass exact versions:

```yaml
jobs:
  mix-check:
    uses: dewetblomerus/actions-elixir/.github/workflows/mix-check.yml@main
    with:
      otp-version: "29.1"
      elixir-version: "1.19.5-otp-29"
```

For a required branch protection check, require the `Mix Check` job.

Examples for the existing apps:

```yaml
# red, because it does not currently have .tool-versions
jobs:
  mix-check:
    uses: dewetblomerus/actions-elixir/.github/workflows/mix-check.yml@main
    with:
      otp-version: "29.1"
      elixir-version: "1.19.5-otp-29"
```

```yaml
# quick-average and mobile-worship can use their checked-in .tool-versions
jobs:
  mix-check:
    uses: dewetblomerus/actions-elixir/.github/workflows/mix-check.yml@main
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
```

## Inputs

Both entry points support these inputs:

| Input | Default | Purpose |
| --- | --- | --- |
| `cache-version` | `v1` | Change this to force fresh Mix caches. |
| `check-command` | `mix check` | Command that must pass before merge. |
| `deps-command` | `mix deps.get` | Dependency installation command. |
| `elixir-version` | `1.19.5-otp-29` | Used only when `version-file` does not exist. |
| `mix-env` | `test` | `MIX_ENV` for dependency installation and checks. |
| `otp-version` | `29.1` | Used only when `version-file` does not exist. |
| `version-file` | `.tool-versions` | Version file read by `erlef/setup-beam`. |
| `working-directory` | `.` | Directory containing `mix.exs`. |

The reusable workflow also supports `runs-on`, defaulting to `ubuntu-latest`.

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
    {:ex_check, "~> 0.16.0", only: [:dev, :test], runtime: false}
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
      "sobelow --config"
    ]
  ]
end
```

Tune the list per repo. For example, apps without Sobelow should remove that task, and apps with asset or Dialyzer checks can add them.

## Notes

This setup follows the useful parts of Felt's Ultimate Elixir CI pattern: shared setup, dependency/build caching, and a single required quality gate. It intentionally leaves the actual quality policy in each app's `mix.exs` so every repository can evolve its checks independently while GitHub Actions stays shared.
