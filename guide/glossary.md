# Glossary (Veteranus)

New terms in GRADUS III. For earlier terms, see the glossaries in
**Tiro** and **Miles**. The lesson where each term first appears is in
brackets.

---

**Action (GitHub)** [03] — A published, reusable CI step, like
`actions/checkout`.

**`act`** [07] — A Testing Library function that makes React finish all
pending updates before your test continues.

**`AbortController`** [07] — A browser object that can cancel a `fetch`
in progress.

**API versioning** [06] — Keeping old and new versions of an API side by
side (e.g. `/v2/…`) so existing clients don't break.

**Audit trail** [05] — An append-only record of changes: what changed,
and when.

**Autouse fixture** [02] — A pytest fixture that runs for every test
without being requested.

**Breaking change** [06] — A change that makes existing clients stop
working, like changing a response's shape.

**Cascade delete** [05] — Deleting child rows automatically when their
parent is deleted (`ON DELETE CASCADE`).

**Cleanup function** [07] — The function an effect returns; React runs
it before the effect runs again and when the component goes away.

**Container** [02] — A running instance of a Docker image.

**Continuous integration (CI)** [03] — Automatically running every check
on a clean machine for every push.

**Coverage threshold** [03] — A minimum coverage percentage that fails
the run when not met.

**Custom hook** [07] — A function starting with `use` that bundles
stateful logic for reuse.

**Data migration** [05] — A migration that changes existing rows rather
than (or as well as) structure.

**Deferred promise** [07] — A promise the test resolves by hand, to
control timing.

**Derived state** [07] — A value calculated from other state during
render, instead of stored separately.

**Docker Compose** [02] — A YAML file describing containers, started
together with `docker compose up`.

**Driver** [02] — The library SQLAlchemy uses to talk to a specific
database (here, psycopg).

**Envelope** [06] — A wrapper object around a list that adds information
like `total`.

**Expression index** [04] — An index on the result of an expression,
such as `lower(name)`.

**Fixture scope** [02] — How often a fixture runs: per test (default),
per module, or once per session.

**Health check** [02] — A command that reports whether a service is
ready, used by Docker and CI.

**Image** [02] — A packaged, read-only snapshot of software, used to
start containers.

**Job (GitHub Actions)** [03] — A group of steps that runs on one fresh
machine.

**Keyset pagination** [06] — Paging by "items after this key" instead of
by offset.

**Limit / offset** [06] — Pagination by "return N rows, after skipping
M."

**Port mapping** [02] — Connecting a port on your machine to a port
inside a container (`host:container`).

**`psql`** [02] — PostgreSQL's command-line client.

**`pydantic-settings`** [02] — A library for typed configuration read
from environment variables and `.env` files.

**Race condition** [07] — A bug where the result depends on the timing
of events you don't control.

**`renderHook`** [07] — Runs a hook inside a hidden test component so
you can test it directly.

**Requirement traceability** [00] — Linking each test to the
requirement it proves.

**Runner** [03] — The machine a CI job runs on.

**Sequence** [02] — The PostgreSQL object that hands out increasing ids.

**Server default** [05] — A column default applied by the database, not
by Python.

**Service container** [03] — A container (like PostgreSQL) started next
to a CI job.

**Test list** [00] — A written list of every test you expect to write,
made before writing them.

**`timestamptz`** [05] — PostgreSQL's time-zone-aware timestamp type.

**Transaction** [02] — A group of database changes that succeed or fail
together, and can be rolled back.

**Volume** [02] — Docker storage that survives when the container is
removed.

**Work breakdown structure (WBS)** [00] — A goal broken down into
smaller pieces until each is a concrete task.

**Workflow** [03] — A GitHub Actions YAML file defining when to run and
what to run.

---

## Latin

| Word | Meaning |
| --- | --- |
| *gradus* | step, rank |
| *tiro* | recruit, beginner |
| *miles* | soldier |
| *veteranus* | veteran (from *vetus*, old, experienced) |
| *bene factum* | well done |
