# 02 — Settings and the test database

**Requirements covered:** INF-1, INF-2, INF-3, INF-4, INF-5, INF-6

**Outcome:** PostgreSQL running in Docker (from memory), a backend that
reads typed settings from the environment or a `.env` file, and a test
suite that builds its database by **running your migrations**.

Scaffold the repo, backend, and frontend first (Miles lesson 01 —
entirely from memory this time, no hints).

---

## Retrieval: Docker, PostgreSQL, and async (INF-1, INF-2, INF-6)

You've done all of this twice. No primer — only the target.

| Setting | Value |
| --- | --- |
| Image | `postgres:17` (pinned) |
| User / password / dev database | `veteranus` / `veteranus` / `veteranus` |
| Test database | `veteranus_test` |
| Host port → container port | **5435** → 5432 |
| Data | a named volume |
| Readiness | a `pg_isready` health check |

- [ ] `compose.yaml` and an init script that creates the test database
- [ ] `docker compose up -d --wait` → **healthy**; `\l` in `psql` lists
      both databases
- [ ] Backend dependencies: FastAPI, `sqlalchemy[asyncio]`, asyncpg,
      Alembic; dev: pytest, pytest-asyncio, pytest-cov, Ruff
- [ ] pytest configured for async tests (auto mode, function-scoped
      loop)
- [ ] Alembic initialized with the **async** template
- [ ] An async engine, an `AsyncSession` factory with
      `expire_on_commit=False`, an async `get_db`, and a `Base` with a
      naming convention (INF-6)

If you need to look something up, the references are Tiro lessons 04–05
and Miles lessons 01–02 — but try first. Every item you look up goes on
your debrief list.

---

## Primer: `pydantic-settings` (INF-3)

In Miles, `config.py` had one function per setting, each calling
`os.getenv`. `pydantic-settings` replaces that with one typed class that
reads environment variables **and** `.env` files, and validates them
like any Pydantic model.

**Example from a different domain** — settings for a weather service:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    forecast_api_url: str = "https://api.example.com"
    request_timeout_seconds: int = 10
```

- Each field is read from the environment variable of the same name,
  case-insensitively (`FORECAST_API_URL`).
- If it's not in the environment, it's read from `.env`.
- If it's in neither, the default is used.
- `REQUEST_TIMEOUT_SECONDS=abc` fails loudly at startup instead of
  causing a strange bug later.

```bash
uv add pydantic-settings
```

Docs: <https://docs.pydantic.dev/latest/concepts/pydantic_settings/> ·
<https://fastapi.tiangolo.com/advanced/settings/>

<details>
<summary>Hint — testing settings</summary>

`monkeypatch.setenv` still works: create a new `Settings()` *after*
setting the variable. To test the `.env` part, `Settings` accepts a
`_env_file` argument pointing at a temporary file — see pytest's
built-in `tmp_path` fixture.
</details>

<details>
<summary>Hint — `.env` and the working directory</summary>

`env_file=".env"` is relative to the folder you **run the command
from**. Run backend commands from `backend/`, and keep `.env` there.
</details>

---

## Primer: a test database built by migrations (INF-4, INF-5)

In Tiro and Miles, an `engine` fixture called `create_all` before every
test. That never tested your **migrations** — a broken migration could
sit behind a green test suite. Now:

1. **Once per test run**, bring the test database's schema up to date
   by running Alembic's `upgrade head` against `TEST_DATABASE_URL`.
2. **Before (or after) every test**, empty the tables.

### Running Alembic from Python

Alembic's commands are also Python functions:

```python
from alembic import command
from alembic.config import Config

config = Config("alembic.ini")
config.set_main_option("sqlalchemy.url", some_url)
command.upgrade(config, "head")
```

Docs: <https://alembic.sqlalchemy.org/en/latest/api/commands.html>

### The async wrinkle

Look at the bottom of your async `migrations/env.py`: online migrations
run through **`asyncio.run(...)`**. `asyncio.run` starts a brand-new
event loop — and Python refuses to start one **while another is already
running**:

```text
RuntimeError: asyncio.run() cannot be called from a running event loop
```

An `async def` fixture or test runs *inside* pytest-asyncio's event
loop. A plain `def` fixture doesn't. So: *what kind of fixture should
call `command.upgrade`?*

<details>
<summary>Hint — Alembic ignores the URL you set</summary>

In Miles, `env.py` *always* overwrote `sqlalchemy.url` with the app's
setting. Now it should respect a URL that was already set (by the
tests), and fall back to the app's setting only if none was given.
Remove the placeholder `sqlalchemy.url` line from `alembic.ini` so
"none was given" is detectable.
</details>

<details>
<summary>Hint — "once per test run"</summary>

Fixtures have a **scope**. `@pytest.fixture(scope="session")` runs once
for the whole run instead of once per test.
<https://docs.pytest.org/en/stable/how-to/fixtures.html#scope-sharing-fixtures-across-classes-modules-packages-or-session>
</details>

<details>
<summary>Hint — the async wrinkle</summary>

Make the migration fixture a **plain, synchronous, session-scoped**
fixture. It runs before any test's event loop exists, so `asyncio.run`
inside `env.py` is free to start its own. Your per-test `async`
fixtures can then depend on it.
</details>

### Emptying tables between tests — two approaches

| Approach | How | Trade-off |
| --- | --- | --- |
| **Truncate** | After each test, run `TRUNCATE <tables> RESTART IDENTITY CASCADE` | Simple and obvious. Slightly slower. |
| **Roll back** | Run each test inside a transaction that's rolled back at the end | Fast and common in large projects. More setup; ids keep climbing. |

Either satisfies INF-5. Pick one, and record why under **Decisions**.

- Truncate with an async connection:
  `await connection.execute(text("TRUNCATE ..."))` inside
  `async with engine.begin() as connection:`.
- The rollback approach works with `AsyncSession` too:
  <https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#joining-a-session-into-an-external-transaction-such-as-for-test-suites>

<details>
<summary>Hint — an autouse fixture</summary>

`@pytest.fixture(autouse=True)` runs for every test without being named
as a parameter — handy for "empty the tables."
</details>

<details>
<summary>Hint — "attached to a different loop" between tests</summary>

Same cause as in Tiro: a pooled connection opened in one test's event
loop, reused in the next. Create the test engine with
`poolclass=NullPool`, or create it per test.
</details>

<details>
<summary>Hint — `%` in a URL</summary>

`set_main_option` treats `%` specially. If a password contains `%`,
escape it as `%%`. (Ours doesn't — but real ones might.)
</details>

---

## Checkpoint

- ✅ `docker compose up -d --wait` → `db` is **healthy** on port 5435
- ✅ Both databases exist (`\l`)
- ✅ `uv run alembic upgrade head` creates tables in the **development**
  database
- ✅ A health test and one database test pass; in `psql`, the **test**
  database has your tables (built by migrations — check
  `alembic_version`), and the **development** database is unchanged
  (INF-2, INF-4)
- ✅ `docker compose down` then `up -d` — your development data is
  still there (INF-1)
- ✅ `backend/.env.example` is committed; `backend/.env` is not

Commit on each green, as always.

Next: [03 — Continuous integration](03-continuous-integration.md)
