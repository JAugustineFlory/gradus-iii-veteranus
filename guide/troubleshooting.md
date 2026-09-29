# Troubleshooting (Veteranus)

Problems new in GRADUS III. For everything set up the same way as
before, see **Tiro's** and **Miles's** `guide/troubleshooting.md` too.

---

## Docker and PostgreSQL

| Symptom | Cause | Fix |
| --- | --- | --- |
| `docker: command not found` or *Cannot connect to the Docker daemon* | Docker Desktop isn't installed or isn't running | Start Docker Desktop and wait until it says it's running |
| `Bind for 0.0.0.0:5432 failed: port is already allocated` | Another PostgreSQL (maybe installed directly) already uses port 5432 | Stop it, or map a different host port: `"5433:5432"`, and use `5433` in your URLs |
| `docker compose ps` never shows **healthy** | Postgres failed to start | `docker compose logs db` and read the last lines |
| The test database doesn't exist, even though you added an init script | Init scripts only run when the volume is **empty** | `docker compose down -v` (deletes all data), then `up -d` |
| Your data vanished after restarting | You ran `down -v`, or the volume isn't mounted | `-v` deletes volumes. Check the `volumes:` section of `compose.yaml` |
| The container keeps restarting after changing the image version | Newer PostgreSQL major versions store data in a different place, or can't read an older version's data | Pin the version you started with. To change versions, `down -v` and start fresh (development data only!) |
| `password authentication failed for user` | The URL's user or password doesn't match `compose.yaml`, or the volume was created with different credentials | Credentials are only applied on first start. Fix the URL, or `down -v` to recreate |
| `ModuleNotFoundError: No module named 'psycopg'` | The driver isn't installed | `uv add "psycopg[binary]"` |
| `Can't load plugin: sqlalchemy.dialects:postgresql.psycopg2` | The URL starts `postgresql://` without a driver name, so SQLAlchemy looks for the older `psycopg2` | Use `postgresql+psycopg://…` |
| `invalid connection option "check_same_thread"` | SQLite-only argument passed to PostgreSQL (INF-6) | Only pass it when the URL is SQLite — or remove it |
| `value too long for type character varying(100)` | PostgreSQL enforces lengths that SQLite ignored | This is PostgreSQL doing its job. Make sure validation (`max_length`) rejects it first with a `422` |

---

## Settings and migrations

| Symptom | Cause | Fix |
| --- | --- | --- |
| Settings ignore your `.env` file | The command ran from a different folder | Run backend commands from `backend/`, where `.env` lives |
| Tests hit the **development** database | Test setup uses `DATABASE_URL`, or Alembic ignores the URL you set | Tests must use `TEST_DATABASE_URL` everywhere; `env.py` must respect a URL already set (lesson 02) |
| `ValueError: invalid interpolation syntax` from Alembic | A `%` in the URL passed to `set_main_option` | Escape it as `%%` |
| Tests pass alone but fail together | Leftover rows from another test, or an assumed `id` | Check your cleanup fixture runs for **every** test (`autouse=True`); compare against returned ids (INF-5) |
| After the migration test, every other test fails | It left the database at an older revision | End it at `head`, even on failure (`try`/`finally` or fixture cleanup) |
| Autogenerate wants to drop and re-create your expression index every time | Alembic can't fully compare expression indexes | Delete those lines from the generated migration. It's a known limitation |
| Backfilled `changed_at` is off by some hours | The date was interpreted in a local time zone | Convert explicitly as UTC (lesson 05 hint) |
| History is in the wrong order for changes made at nearly the same time | `now()` is the transaction's start time, so timestamps can tie | Order by `changed_at`, then `id` |

---

## CI

| Symptom | Cause | Fix |
| --- | --- | --- |
| The workflow never runs | The file isn't at `.github/workflows/<name>.yml`, or the YAML is invalid | Check the path; paste the YAML into the VS Code GitHub Actions extension to see errors |
| `connection refused` to PostgreSQL in CI | Wrong host/port, or tests started before Postgres was ready | Use `localhost` and the mapped port; keep the health-check `options` |
| `npm ci` fails: lockfile out of sync | `package.json` changed without updating `package-lock.json` | Run `npm install` locally and commit the lockfile |
| `uv sync --locked` fails | `pyproject.toml` changed without updating `uv.lock` | Run `uv lock` (or `uv sync`) locally and commit `uv.lock` |
| Passes locally, fails in CI | Something only exists on your machine: an uncommitted file, a `.env` value, an installed tool | Read the failing step's log; do the fresh-clone check |
| Vitest: *No test files found* | No tests yet | Write at least one test (lesson 03) |
| Coverage threshold fails in CI but not locally | CI runs a different command (e.g. with coverage) | Run exactly the CI command locally |

---

## Hooks and race conditions

| Symptom | Cause | Fix |
| --- | --- | --- |
| *Invalid hook call* | A hook was called outside a component or custom hook, or conditionally | Only call hooks at the top level of components and `use…` functions |
| `result.current` has old values in a hook test | You stored `result.current` in a variable before the update | Read `result.current` fresh after every change |
| *not wrapped in act(...)* warnings in hook tests | A state update happened outside `act` | Wrap resolving promises and calling actions in `act`, or wait with `waitFor` |
| The UI-5 test passes even without the stale-response guard | Both promises resolve in the order they were created, or at once | Resolve the **second** first, inside its own `act`, then the first |
| ESLint: *Calling setState synchronously within an effect* | Setting state (like `loading`) directly in an effect body | Derive the value instead (lesson 07), or set it in the event handler that triggers the change |
| The list briefly shows the wrong page after changing the filter | The page is reset in an effect that runs after the fetch for the old page | Reset the page in the filter's change handler (lesson 06) |

---

## Still stuck?

1. Re-read the requirement's exact wording — many bugs are a
   misreading.
2. Shrink the problem: write the smallest possible test that shows it.
3. Search the exact error message in the official docs linked in the
   lesson.
4. Check your Miles code for how you solved the nearest similar
   problem.
