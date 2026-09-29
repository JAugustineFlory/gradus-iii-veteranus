# 04 — Rebuild the core

**Requirements covered:** CO-1, CO-2, CO-3, AP-1, AP-2, AP-3, AP-5,
AP-6, AP-7, UI-1

**Outcome:** everything Miles did, rebuilt on PostgreSQL — this time
with tests you wrote.

This lesson is almost entirely **retrieval**. You've built all of it
before. The new parts are writing every test yourself and one
PostgreSQL-flavored requirement (CO-3).

---

## One temporary shortcut

AP-4 changes `GET /applications` from returning a plain list to
returning a **page** object. Build the list as a plain array for now,
exactly like Miles. Lesson 06 changes it — on purpose, because changing
an API that a frontend already depends on is a skill of its own.

---

## Writing your own tests

Your test list from lesson 00 is the plan. Work through it one line at
a time: write the test, watch it fail, make it pass, cross it off.

**You may read your Miles tests** to remember *how* something is tested
(the `client` fixture, `vi.mock`, `findBy…`). **Don't copy them.**
Writing a test from a requirement is the skill Veteranus exists to
build — and your Miles tests assumed things Veteranus forbids, like
`"id": 1` (INF-5).

<details>
<summary>Hint — asserting without assuming ids</summary>

Compare against the id the API *gave you*:

```python
created = create_company(client, "Acme")
response = client.get(...)
assert response.json()["company"]["id"] == created["id"]
```

Or leave `id` out of the comparison and check the other fields.
</details>

<details>
<summary>Hint — test helpers</summary>

Miles repeated `create_company` in several files. A **fixture that
returns a function** can live in `conftest.py` and be shared:

```python
@pytest.fixture
def make_widget(client):
    def _make(name="Sprocket"):
        ...
    return _make
```
</details>

---

## Primer: case-insensitive comparison (CO-3)

PostgreSQL compares strings **exactly**: `'Acme' = 'acme'` is false.
To compare ignoring case, lower-case both sides.

**Example from a different domain** — finding a user by email, where
`Sam@Example.com` and `sam@example.com` should match:

```python
select(User).where(func.lower(User.email) == email.lower())
```

- **`func.lower(...)`** generates the SQL function `lower(...)`, run by
  the database on the column.
- **`email.lower()`** is Python's string method, run on the value you
  already have.

Docs:
<https://docs.sqlalchemy.org/en/20/core/sqlelement.html#sqlalchemy.sql.expression.func>

### Going further (a "should", not a "must")

Your check-before-insert guards the API, but the database's plain
`unique` constraint still allows `Acme` and `acme` side by side if
something bypasses the API. PostgreSQL can enforce case-insensitive
uniqueness with a **unique index on `lower(name)`** — an *expression
index*.

<details>
<summary>Hint</summary>

SQLAlchemy's `Index` accepts expressions like `func.lower(...)` and a
`unique=True` flag. Alembic's autogenerate may not render expression
indexes perfectly: read the migration, and edit it if needed.
<https://docs.sqlalchemy.org/en/20/core/constraints.html#functional-indexes>
</details>

If you skip it, record that under **Decisions**.

---

## Frontend

Rebuild the Miles UI to satisfy UI-1: types, `formatStatus`, the API
module with a generic `request`, `StatusFilter`, `CompanyForm`,
`ApplicationList`, `ApplicationForm`, and `App` — each test-first.

For now `App` may call the API module directly, as in Miles. Lesson 07
moves that into custom hooks (UI-6), and your tests will tell you
whether the move broke anything.

---

## Checkpoint

- ✅ Every CO and AP requirement (except AP-4) has passing tests with
  its ID in the name
- ✅ UI-1 tests pass
- ✅ Coverage thresholds met; CI green
- ✅ In `psql`: `\d companies` and `\d applications` show your columns,
  constraints, and the foreign key with their convention-based names

Next: [05 — Status history and data migration](05-status-history.md)
