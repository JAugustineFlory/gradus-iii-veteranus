# 02 — Docker and PostgreSQL

**Requirements covered:** INF-1, INF-2, INF-3, INF-4, INF-5, INF-6

**Outcome:** PostgreSQL running in Docker; a backend that reads typed
settings from the environment or a `.env` file; and a test suite that
builds its database by running your migrations.

Scaffold the repo, backend, and frontend first (Miles lesson 01 —
entirely from memory this time).

---

## Primer: Docker

**The problem Docker solves.** Installing PostgreSQL directly on your
machine means a different install on Windows, macOS, and Linux, version
clashes between projects, and "works on my machine" bugs. Docker runs
software in an isolated, identical box on every machine.

| Term | Plain English |
| --- | --- |
| **Image** | A packaged, read-only snapshot of software and everything it needs — e.g. `postgres:17`. Downloaded from Docker Hub. |
| **Container** | A running instance of an image. You can start, stop, and delete it; the image is unchanged. |
| **Volume** | Storage that lives *outside* the container, so data survives when the container is deleted. |
| **Port mapping** | `"5432:5432"` connects port 5432 on *your* machine to port 5432 *inside* the container. Format: `host:container`. |
| **Docker Compose** | A YAML file (`compose.yaml`) describing one or more containers, so `docker compose up` starts everything the same way every time. |

**Install:** Docker Desktop from <https://docs.docker.com/get-started/get-docker/>.
On Windows it requires WSL 2; the installer walks you through it.

**Check:**

```bash
docker --version
docker compose version
docker run hello-world
```

✅ The last one prints *Hello from Docker!*

### A PostgreSQL service in Compose

This is new infrastructure, so here's a complete, annotated example.
Put it at the repo root as `compose.yaml`:

```yaml
services:
  db:
    image: postgres:17          # pin a major version
    environment:
      POSTGRES_USER: veteranus
      POSTGRES_PASSWORD: veteranus
      POSTGRES_DB: veteranus    # the development database
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U veteranus"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  pgdata:
```

- **Pin the version.** `postgres:latest` changes under you. (Newer
  major versions also change where data is stored inside the container,
  which breaks the volume path above.)
- **`environment`** — the official image reads these on **first
  start** to create a user and a database.
- **`volumes`** — the named volume `pgdata` holds the data (INF-1).
- **`healthcheck`** — lets Docker (and later, CI) know when Postgres is
  actually ready to accept connections, not just started.
- These credentials are for a **local development** database only.
  Never reuse them anywhere real.

```bash
docker compose up -d        # -d: run in the background
docker compose ps           # STATUS should become "healthy"
docker compose logs db      # see what Postgres is saying
docker compose down         # stop and remove the container (volume kept)
docker compose down -v      # …and delete the volume (all data!)
```

Docs: <https://docs.docker.com/compose/gettingstarted/> ·
<https://hub.docker.com/_/postgres>

### Talking to PostgreSQL directly

`psql` is PostgreSQL's command-line client. It's already inside the
container:

```bash
docker compose exec db psql -U veteranus -d veteranus
```

| Command | Does |
| --- | --- |
| `\l` | list databases |
| `\dt` | list tables |
| `\d applications` | describe a table |
| `SELECT * FROM applications;` | query (end with `;`) |
| `\q` | quit |

### Your task: INF-2 (the test database)

The Compose file above creates only the development database. You need
a second one for tests.

<details>
<summary>Hint 1</summary>

Read the **Initialization scripts** section of the Postgres image's
Docker Hub page. Files mounted into a particular folder run once, when
the database is first created.
</details>

<details>
<summary>Hint 2</summary>

A one-line `.sql` file with a `CREATE DATABASE` statement, mounted from a
folder in your repo. Init scripts only run on a **fresh** volume — after
adding one, `docker compose down -v` and `up -d` again.
</details>

**Checkpoint:** in `psql`, `\l` lists both databases.

---

## Primer: PostgreSQL from Python

### The driver

SQLAlchemy doesn't talk to PostgreSQL itself; it uses a **driver**.
Install **psycopg** (version 3):

```bash
uv add "psycopg[binary]"
```

### Connection URLs

```text
postgresql+psycopg://veteranus:veteranus@localhost:5432/veteranus
└──┬─────┘ └──┬──┘   └──┬────┘ └──┬────┘ └──┬────┘ └┬─┘ └──┬────┘
 dialect   driver     user    password    host   port  database
```

### Differences from SQLite that will bite you

- **Real types.** PostgreSQL enforces column types and lengths strictly.
  A string too long for `String(100)` is an error, not silently stored.
- **Foreign keys are always enforced.** Your Miles `PRAGMA` listener is
  SQLite-only (INF-6).
- **`check_same_thread`** is a SQLite-only connection argument.
  Passing it to PostgreSQL is an error (INF-6).
- **ids come from sequences**, which don't rewind when you roll back or
  delete. That's one reason for INF-5's "no assumed ids."
- **Batch mode** in Alembic isn't needed for PostgreSQL. Leaving it on
  is harmless.

Docs:
<https://docs.sqlalchemy.org/en/20/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.psycopg>

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

In Miles, each test created tables with `create_all` on a throwaway
in-memory database. That never tested your **migrations**, and
PostgreSQL has no in-memory mode. Now:

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

### Emptying tables between tests — two approaches

| Approach | How | Trade-off |
| --- | --- | --- |
| **Truncate** | After each test, run `TRUNCATE <tables> RESTART IDENTITY CASCADE` | Simple and obvious. Slightly slower. |
| **Roll back** | Run each test inside a transaction that's rolled back at the end | Fast and common in large projects. More setup; ids keep climbing. |

Either satisfies INF-5. Pick one, and record why under **Decisions**.

Docs for the rollback approach:
<https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#joining-a-session-into-an-external-transaction-such-as-for-test-suites>

<details>
<summary>Hint — an autouse fixture</summary>

`@pytest.fixture(autouse=True)` runs for every test without being named
as a parameter — handy for "empty the tables."
</details>

<details>
<summary>Hint — `%` in a URL</summary>

`set_main_option` treats `%` specially. If a password contains `%`,
escape it as `%%`. (Ours doesn't — but real ones might.)
</details>

---

## Checkpoint

- ✅ `docker compose up -d` → `db` is **healthy**
- ✅ Both databases exist (`\l`)
- ✅ `uv run alembic upgrade head` creates tables in the **development**
  database
- ✅ A health test and one database test pass, and afterward the
  **development** database is unchanged (INF-2)
- ✅ `docker compose down` then `up -d` — your development data is
  still there (INF-1)
- ✅ `backend/.env.example` is committed; `backend/.env` is not

Commit on each green, as always.

Next: [03 — Continuous integration](03-continuous-integration.md)
