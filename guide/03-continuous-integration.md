# 03 — Continuous integration

**Requirements covered:** INF-7, INF-8, INF-9

**Outcome:** every push and pull request runs your checks on GitHub's
machines, and the README shows whether `main` is passing.

Set this up **now**, before the features. From here on, CI checks every
commit you push — that's the point.

---

## Primer: why CI

Your Husky hook checks commits on *your* machine. But:

- hooks can be skipped (`--no-verify`) or not installed;
- "passes on my machine" can hide a missing file or a setting only you
  have;
- a teammate's pull request should be checked before anyone merges it.

**Continuous integration** (CI) runs the checks on a clean machine, from
a fresh clone, on every push. If it passes there, it's not an accident
of your setup. It's the fresh-clone test from Tiro lesson 08 — automated
and run every time.

---

## Primer: GitHub Actions

| Term | Plain English |
| --- | --- |
| **Workflow** | A YAML file in `.github/workflows/`. Describes *when* to run and *what* to run. |
| **Trigger** (`on`) | The events that start it: `push`, `pull_request`, a schedule… |
| **Job** | A group of steps that runs on one fresh virtual machine. Jobs run in parallel unless told otherwise. |
| **Runner** | The machine a job runs on, e.g. `ubuntu-latest`. |
| **Step** | One command (`run:`) or one reusable **action** (`uses:`). |
| **Action** | A published, reusable step — e.g. `actions/checkout` clones your repo onto the runner. |
| **Service container** | A Docker container started alongside a job — e.g. PostgreSQL for your tests. |

Docs: <https://docs.github.com/en/actions/writing-workflows/quickstart>

### A skeleton

This is new, so here's the structure, with the steps you must fill in
marked `TODO`. Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  backend:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: veteranus
          POSTGRES_PASSWORD: veteranus
          POSTGRES_DB: veteranus_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U veteranus"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10

    env:
      TEST_DATABASE_URL: TODO   # point at the service's database

    defaults:
      run:
        working-directory: backend

    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
      # TODO: install the locked dependencies
      # TODO: lint and format check
      # TODO: run the tests

  frontend:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: frontend

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      # TODO: install the locked dependencies
      # TODO: lint, type-check, and test
```

- **`services.postgres`** starts PostgreSQL next to the job, reachable
  at `localhost:5432`. Its database is created directly as the *test*
  database — CI never needs a development database.
- **`options`** with the health check makes the job wait until Postgres
  is ready.
- **`defaults.run.working-directory`** means every `run:` step starts in
  that folder.
- **Action versions** (`@v4`, `@v6`) are major versions. Check each
  action's page on the GitHub Marketplace for the latest major version.

<details>
<summary>Hint — installing locked dependencies</summary>

Locally you used `uv sync` and `npm install`. In CI, prefer the command
that installs **exactly** what's in the lockfile and fails if the
lockfile is out of date: `npm ci` for Node. For uv, look up the
`--locked` flag on `uv sync`.
</details>

<details>
<summary>Hint — the workflow runs but can't find Postgres</summary>

In a service container setup where the job runs directly on the runner
(not in its own container), the host is `localhost` and the port is the
mapped one.
</details>

### The status badge

GitHub generates a badge image for every workflow. On your repo's
**Actions** tab, open the workflow, find the **⋯** menu, and choose
**Create status badge**. Paste the Markdown at the top of your README.

---

## Primer: coverage thresholds (INF-8)

A threshold turns coverage from a number you *look at* into a rule the
build *enforces*.

- **pytest-cov:** there's a command-line option that fails the run below
  a percentage. Add it to `addopts`.
  <https://pytest-cov.readthedocs.io/en/latest/config.html>
- **Vitest:** the `coverage` config accepts `thresholds`.
  <https://vitest.dev/config/#coverage-thresholds>

For CI to enforce the frontend threshold, the frontend test step must
run **with coverage**.

> **Early on, the frontend has almost no code** — and Vitest fails when
> it finds no tests at all. Write one small test now (for example, that
> `App` renders the **Job tracker** heading) so both the test run and
> the threshold have something to measure.

---

## Checkpoint

- ✅ Push, open the **Actions** tab: both jobs go green
- ✅ The badge in your README shows **passing**
- ✅ **Break it on purpose:** push a commit with a failing test (use
  `git commit --no-verify` to get past your own hook). CI goes red. Fix
  it and push again; CI goes green.
- ✅ Temporarily lower a test's reach so coverage drops below the
  threshold — the run fails. Put it back.

Next: [04 — Rebuild the core](04-rebuild-the-core.md)
