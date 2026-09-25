# ci-workflows

Shared GitHub Actions workflows for Python and AI-agent projects. Fix a bug
here once and every project inherits it.

## Testing layers

Agent output is non-deterministic and real model calls cost money, so tests
are split by cadence rather than run all at once.

| Layer | What | Runs | Cost | Where |
| --- | --- | --- | --- | --- |
| 1 | Unit tests with mocks | Every push | Free | `python-test.yml` |
| 2 | Recorded HTTP replay (`vcrpy` / `respx`) | Every push | Free | `python-test.yml` |
| 3 | Smoke tests against real APIs | Nightly | Cents | `agent-eval.yml` |
| 4 | Golden-dataset evals, LLM-as-judge | Nightly / pre-release | Dollars | `agent-eval.yml` |

Layers 1–2 gate merges. Layers 3–4 report; a failure means *look at this*,
not *the build is broken*.

## Usage

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "0 3 * * *"        # nightly evals

jobs:
  test:
    uses: siopinto/ci-workflows/.github/workflows/python-test.yml@v1
    with:
      pytest-args: '-m "not integration"'

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: siopinto/ci-workflows/.github/workflows/docker-deploy.yml@v1
    with:
      deploy-dir: /home/mysio/my-project
      health-url: http://127.0.0.1:8000/health

  eval:
    if: github.event_name == 'schedule'
    uses: siopinto/ci-workflows/.github/workflows/agent-eval.yml@v1
    secrets: inherit
```

## Workflows

### `python-test.yml`

Installs from the lock file so CI runs the versions production runs, then
runs pytest with coverage. Deliberately receives **no secrets** — anything
needing a real key belongs in `agent-eval.yml`.

| Input | Default | Notes |
| --- | --- | --- |
| `python-version` | `3.12` | |
| `requirements-file` | `requirements.lock.txt` | |
| `dev-requirements-file` | `requirements-dev.txt` | |
| `pytest-args` | `""` | Use `-m "not integration"` to exclude real-API tests |
| `working-directory` | `.` | For monorepos |
| `run-lint` | `false` | ruff check + format --check |
| `upload-coverage` | `false` | |

### `docker-deploy.yml`

For hosts behind NAT: a self-hosted runner polls GitHub outbound, so no
inbound ports or SSH exposure. Builds on the target, so no registry auth and
no architecture mismatch. Rolls back to the previous commit if the health
gate or smoke test fails.

| Input | Default | Notes |
| --- | --- | --- |
| `deploy-dir` | *required* | Absolute path to the checkout on the host |
| `health-url` | `http://127.0.0.1:8000/health` | |
| `health-timeout` | `30` | Seconds |
| `compose-file` | `docker-compose.yml` | |
| `runner-labels` | `'["self-hosted","Linux"]'` | JSON array |
| `smoke-command` | `""` | Shell command; non-zero triggers rollback |
| `prune-images` | `true` | |

⚠️ The deploy step runs `git reset --hard`, which **discards uncommitted
changes** in the deploy directory. Untracked files such as `.env` are
unaffected.

### `agent-eval.yml`

Runs pytest tests carrying a marker (default `integration`) and/or a
promptfoo config. Both optional. Sets `LANGSMITH_TRACING` automatically when
`LANGSMITH_API_KEY` is present, so a regressed eval has a full trace of which
tools were called and what they returned.

| Input | Default | Notes |
| --- | --- | --- |
| `pytest-marker` | `integration` | Empty to skip |
| `promptfoo-config` | `""` | Path to `promptfooconfig.yaml`; empty to skip |
| `fail-on-eval-regression` | `true` | `false` to report without failing |

Pass credentials with `secrets: inherit`.

## Composite actions

### `actions/health-check`

Poll a URL until it succeeds. Useful inside a job you write yourself.

```yaml
- uses: siopinto/ci-workflows/actions/health-check@v1
  with:
    url: http://127.0.0.1:8000/health
    timeout: "60"
```

## Versioning

Consumers should pin a tag, never `@main`:

```yaml
uses: siopinto/ci-workflows/.github/workflows/python-test.yml@v1
```

If every project tracked `@main`, one bad commit here would break every
pipeline at once — including the deploy needed to fix it.

Moving the tag after a change:

```bash
git tag -fa v1 -m "v1" && git push --force origin v1
```

## If this repo is private

Other private repos cannot call these workflows until you enable
**Settings → Actions → General → Access →
"Accessible from repositories owned by the user"** here. Without it, callers
fail with a confusing *workflow not found*. Making the repo public sidesteps
it — workflow definitions contain no secrets.
