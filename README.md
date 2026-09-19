# Shared reusable GitHub Actions workflows.

One place for CI logic, instead of copy-pasting YAML into every repo.

## Workflows

| Workflow | Purpose |
| -------- | ------- |
| `.github/workflows/python-ci.yml` | Standard CI for uv-based Python repos: checkout, install uv, install Python, `uv sync`, lint, test. |

## Usage

The caller repo keeps a tiny workflow that handles triggers and delegates:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: IAmNo1Special/actions/.github/workflows/python-ci.yml@v1
```

### Inputs

| Input | Default | Purpose |
| ----- | ------- | ------- |
| `python-version` | `"3.13"` | Python version installed via `uv python install` |
| `working-directory` | `"."` | Directory the install/lint/test steps run in |
| `pre-commands` | `""` | Shell commands run after checkout, before install (e.g. `git clone` an extra repo) |
| `install-command` | `"uv sync"` | Dependency install command (e.g. `"uv sync --all-packages --frozen"`) |
| `run-lint` | `true` | Whether to run the lint step |
| `lint-commands` | ruff check + format check + mypy | Lint/typecheck commands (multiline shell) |
| `run-tests` | `true` | Whether to run the test step |
| `test-commands` | `uv run python -m pytest -q -p no:cacheprovider` | Test commands (multiline shell) |
| `coverage-artifact-name` | `""` | When set, upload coverage data as an artifact with this name after tests |
| `coverage-artifact-path` | `".coverage.*"` | Files to upload as the coverage artifact (relative to repo root) |
| `test-env-json` | `"{}"` | JSON object of env vars for the test step, e.g. `'{"HUB_SECRET_KEY": "test-value"}'` |
| `runs-on` | `"ubuntu-latest"` | Runner label |
| `timeout-minutes` | `20` | Job timeout |

### Examples

Custom test command with extra env (Soulscape-style):

```yaml
jobs:
  ci:
    uses: IAmNo1Special/actions/.github/workflows/python-ci.yml@v1
    with:
      install-command: "uv sync --frozen"
      run-lint: false
      test-env-json: '{"HUB_SECRET_KEY": "soulscape-secret-123"}'
      test-commands: |
        uv run --package server python -m pytest server/tests/ -q -p no:cacheprovider
        uv run --package client python -m pytest client/tests/unit -q -p no:cacheprovider
```

Repo needing an extra checkout first (marketplace-style):

```yaml
jobs:
  ci:
    uses: IAmNo1Special/actions/.github/workflows/python-ci.yml@v1
    with:
      pre-commands: |
        git clone --depth 1 https://github.com/IAmNo1Special/mvgeos mvgeos
      install-command: "cd mvgeos && uv sync --all-packages"
      run-lint: false
      test-commands: |
        uv run --project mvgeos python -m pytest runes/ -q
```

Matrix over Python versions:

```yaml
jobs:
  ci:
    strategy:
      matrix:
        python-version: ["3.12", "3.13"]
    uses: IAmNo1Special/actions/.github/workflows/python-ci.yml@v1
    with:
      python-version: ${{ matrix.python-version }}
```

## Versioning

Releases are tags (`v1`, `v2`, ...). Callers pin a major tag; breaking
changes bump the major tag so existing callers are never surprised.

## Notes

- Secrets are intentionally not passed through: keep CI hermetic (mocked
  seams, no real credentials), like the repos this was extracted from.
- `test-env-json` is for non-secret test env vars only.
