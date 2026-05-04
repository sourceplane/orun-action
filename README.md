# orun-action

The official GitHub Action for [orun](https://github.com/sourceplane/orun) — install orun, compile execution plans, and fan out jobs across GitHub Actions matrix runners.

## Sub-actions

| Action | Purpose |
|---|---|
| `sourceplane/orun-action@v1` | Install orun and add to PATH |
| `sourceplane/orun-action/plan@v1` | Compile `intent.yaml` → `plan.json`, emit job-ids matrix |
| `sourceplane/orun-action/run@v1` | Execute a single job from a compiled plan |
| `sourceplane/orun-action/validate@v1` | Validate `intent.yaml` (fast PR gate) |

## Quick start

### PR validation gate

```yaml
on: [pull_request]
permissions:
  contents: read
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: sourceplane/orun-action/validate@v1
        with:
          intent: intent.yaml
```

### GHA-native matrix execution (recommended for production)

Compile the plan once, then execute each job in its own GitHub Actions runner in parallel. Dependency ordering is enforced by the orun remote state backend.

```yaml
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write   # Remote state OIDC authentication

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      job-ids: ${{ steps.plan.outputs.job-ids }}
      plan-checksum: ${{ steps.plan.outputs.plan-checksum }}
    steps:
      - uses: actions/checkout@v4
      - uses: sourceplane/orun-action/plan@v1
        id: plan
        with:
          intent: intent.yaml

  execute:
    needs: plan
    if: needs.plan.outputs.job-ids != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      max-parallel: 10
      matrix:
        job-id: ${{ fromJson(needs.plan.outputs.job-ids) }}
    concurrency:
      group: orun-${{ github.ref }}-${{ matrix.job-id }}
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4
      - uses: sourceplane/orun-action/run@v1
        with:
          plan: plan.json
          job: ${{ matrix.job-id }}
          download-artifact: true
          remote-state: true
          backend-url: ${{ vars.ORUN_BACKEND_URL }}
```

## Inputs

### `sourceplane/orun-action@v1` (setup)

| Input | Default | Description |
|---|---|---|
| `version` | `latest` | orun version tag or `latest` (auto-resolves to newest release) |
| `install-dir` | `~/.local/bin` | Directory to install the binary |
| `install-url` | upstream | Override install script URL (air-gapped) |

**Output:** `version` — resolved version string.

### `sourceplane/orun-action/plan@v1`

| Input | Required | Default | Description |
|---|---|---|---|
| `intent` | yes | — | Path to `intent.yaml` |
| `version` | — | `latest` | orun version |
| `output` | — | `plan.json` | Output path |
| `changed-only` | — | `false` | Scope to changed components |
| `base-ref` | — | `""` | Base SHA for changed-only |
| `head-ref` | — | `""` | Head SHA for changed-only |
| `upload-artifact` | — | `true` | Upload plan as workflow artifact |
| `artifact-name` | — | `orun-plan` | Artifact name |

**Outputs:** `plan-file`, `plan-checksum`, `job-ids` (JSON array), `job-count`.

### `sourceplane/orun-action/run@v1`

| Input | Required | Default | Description |
|---|---|---|---|
| `plan` | yes | — | Path to `plan.json` |
| `job` | — | `""` | Job ID (omit for sequential all) |
| `version` | — | `latest` | orun version |
| `runner` | — | `github-actions` | Execution backend |
| `remote-state` | — | `false` | Enable distributed coordination |
| `backend-url` | — | `""` | orun backend URL |
| `exec-id` | — | `run_id-attempt` | Stable execution ID |
| `dry-run` | — | `false` | Print without executing |
| `download-artifact` | — | `false` | Download plan from artifact |
| `artifact-name` | — | `orun-plan` | Artifact name to download |

**Outputs:** `exec-id`, `status`.

### `sourceplane/orun-action/validate@v1`

| Input | Required | Description |
|---|---|---|
| `intent` | yes | Path to `intent.yaml` |
| `version` | — | orun version |

**Output:** `valid` — `"true"` if validation passed.

## Permissions

| Operation | `contents` | `id-token` |
|---|---|---|
| validate only | `read` | — |
| plan only | `read` | — |
| plan + run (local) | `read` | — |
| plan + run (remote state) | `read` | `write` |

## Version pinning

```yaml
# Floating major (convenience — always gets latest orun)
- uses: sourceplane/orun-action/plan@v1

# Pinned semver (recommended)
- uses: sourceplane/orun-action/plan@v1.1.0

# Pinned SHA (high-assurance / supply-chain audited)
- uses: sourceplane/orun-action/plan@<commit-sha>  # v1.1.0
```

The `v1` floating tag is force-pushed on every patch and minor release. Major version increments only on breaking input/output changes.

## Caching

The orun binary is cached by `runner.os`, `runner.arch`, and the resolved version tag. When `version: latest` is used, the action resolves the actual release tag first so the cache key stays stable until a new orun release is published.

Optionally cache composition archives:

```yaml
- uses: actions/cache@v4
  with:
    path: .orun/cache
    key: orun-compositions-${{ hashFiles('.orun/compositions.lock.yaml') }}
```

## Log output

All actions use GitHub Actions [log grouping](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#grouping-log-lines) for clean, collapsible output. Key events are surfaced as `::notice::` annotations and failures use `::error::` annotations with file-level targeting where applicable.

Each action writes a structured [job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary) with execution results.
