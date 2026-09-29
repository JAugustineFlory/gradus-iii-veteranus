# 05 — Status history and data migration

**Requirements covered:** HI-1 through HI-6, UI-4

**Outcome:** every status change is recorded; history appears in the
UI; and the migration that introduced history filled it in for
applications that already existed — proven by a test.

---

## Primer: audit trails

Right now an application knows only its **current** status. You can't
answer "when did I get the interview?" or "how long between applying and
the offer?" An **audit trail** — an append-only log of changes — answers
both. The internship's applicant tracker will almost certainly need one:
who moved an applicant to which stage, and when.

The rule of an audit trail: rows are **added**, never edited.

---

## Primer: cascade deletes enforced by the database (HI-5)

When a parent row is deleted, what happens to rows that point at it?
With a foreign key, the database refuses the delete by default —
children would be left pointing at nothing.

**`ON DELETE CASCADE`** tells the database: delete the children too.

**Example from a different domain** — a blog, where deleting a post
deletes its comments:

```python
class Comment(Base):
    __tablename__ = "comments"
    ...
    post_id: Mapped[int] = mapped_column(
        ForeignKey("posts.id", ondelete="CASCADE"),
    )


class Post(Base):
    __tablename__ = "posts"
    ...
    comments: Mapped[list["Comment"]] = relationship(
        back_populates="post",
        passive_deletes=True,
    )
```

- **`ondelete="CASCADE"`** goes into the table definition — and so into
  your migration. The **database** enforces it, whoever deletes the row.
- **`passive_deletes=True`** tells SQLAlchemy "the database handles
  this; don't load every comment first just to delete it."

HI-5 says "enforced by the database." How could a test prove the
*database* did it, rather than your Python code?

<details>
<summary>Hint</summary>

Delete the application with a **raw SQL statement** (bypassing the ORM
entirely), then count its history rows.
</details>

Docs:
<https://docs.sqlalchemy.org/en/20/orm/cascades.html#using-foreign-key-on-delete-cascade-with-orm-relationships>

---

## Primer: timestamps set by the database (HI-1)

```python
changed_at: Mapped[datetime] = mapped_column(
    DateTime(timezone=True),
    server_default=func.now(),
)
```

- **`timezone=True`** creates a PostgreSQL `timestamptz` column, which
  stores an exact moment. Without it, `10:00` is ambiguous — whose
  10:00?
- **`server_default=func.now()`** means the *database* fills it in, so
  every row gets a consistent clock no matter which server inserted it.

One PostgreSQL detail: **`now()` returns the time the current
transaction started**, so rows inserted in the same transaction share a
timestamp. "Oldest first" (HI-4) therefore needs a **tie-breaker** in
the ordering.

<details>
<summary>Hint — the tie-breaker</summary>

`id` always increases. Order by `changed_at`, then by `id`.
</details>

Docs: <https://docs.sqlalchemy.org/en/20/core/defaults.html#server-invoked-ddl-explicit-default-expressions>

---

## Where should the "record a change" code live? (HI-2, HI-3)

Three reasonable places:

| Option | Pros | Cons |
| --- | --- | --- |
| **In the route handlers**, next to the create/update code | Explicit and easy to read and test | Every future code path that changes status must remember to do it |
| **A SQLAlchemy event** that fires when `status` changes | Can't be forgotten by Python code | "Spooky action at a distance"; harder to follow |
| **A database trigger** | Nothing can bypass it, even raw SQL | Logic lives outside your Python code and tests |

Any of them can satisfy the requirements. Choose one and record it under
**Decisions**.

<details>
<summary>Hint — detecting "the status actually changed"</summary>

You already compute the fields the client sent (`exclude_unset`). The
status changed only if `status` is among them **and** differs from the
current value — compare *before* applying the changes.
</details>

---

## Primer: data migrations (HI-6)

So far, every migration changed the **structure**. A **data
migration** changes the **rows**.

HI-6 says: when `status_changes` is created, every existing application
must get its starting row. Without that, applications created *before*
the feature have no history at all — and the UI would show nothing for
them.

### The golden rule

**Never import your app's models into a migration.** Models describe
the schema *today*. A migration must work against the schema *as it was
when that migration was written* — and it must keep working a year from
now, after the models have changed ten times.

Use plain SQL (`op.execute`) or lightweight table definitions
(`sa.table(...)`) that exist only inside the migration.

### Example from a different domain

A migration that adds a `notifications` table, then gives every existing
user a welcome notification:

```python
def upgrade():
    op.create_table(
        "notifications",
        # … columns …
    )
    op.execute(
        """
        INSERT INTO notifications (user_id, message, created_at)
        SELECT id, 'Welcome!', signed_up_on
        FROM users
        """
    )


def downgrade():
    op.drop_table("notifications")
```

- **`INSERT … SELECT`** copies from one table into another in a single
  statement, inside the database. No rows travel to Python and back —
  fast even for millions of rows.
- The downgrade drops the table; the backfilled rows were derived from
  other data, so nothing is lost.

<details>
<summary>Hint — `applied_on` at midnight UTC</summary>

`applied_on` is a `date`; `changed_at` is a `timestamptz`. In
PostgreSQL, casting a date to `timestamp` gives midnight, and
`AT TIME ZONE 'UTC'` turns a plain timestamp into a `timestamptz`
interpreted as UTC. Look up both in the PostgreSQL docs:
<https://www.postgresql.org/docs/current/functions-datetime.html>
</details>

Docs: <https://alembic.sqlalchemy.org/en/latest/ops.html#alembic.operations.Operations.execute>

---

## Primer: testing a migration (HI-6)

Your normal tests start from `upgrade head` with empty tables — so they
never see a migration handle **existing** data. A migration test does
this instead:

```text
1. downgrade the test database to base (empty)
2. upgrade to the revision JUST BEFORE the one under test
3. insert rows with raw SQL, shaped like the schema at THAT point
4. upgrade to the revision under test
5. assert the new rows are correct
6. upgrade to head, so later tests find the schema they expect
```

- Revision ids are in each migration file (`revision = "…"`). Hard-code
  the two you need in the test.
- Use raw SQL in step 3 for the same reason as the golden rule: your
  models describe *today's* schema, not the one at step 2.

<details>
<summary>Hint — running raw SQL in a test</summary>

```python
with engine.begin() as connection:
    connection.execute(text("INSERT …"), {"name": "Acme"})
```

`begin()` commits automatically when the block ends. `text` comes from
`sqlalchemy`; `:name` placeholders in the SQL are filled from the
dictionary.
</details>

<details>
<summary>Hint — this test fights with my cleanup fixture</summary>

Your per-test cleanup expects the schema to be at `head`. Make sure the
migration test *ends* at head (step 6), even if an assertion fails —
think `try` / `finally`, or a fixture with cleanup after `yield`.
</details>

---

## Frontend (UI-4)

- An API function for `GET /applications/{id}/history`.
- A **History** button per application that shows and hides the list.
  Load the history when it's first opened.
- Entries: **Created as Applied** for the first; **Interviewing →
  Offer** for later ones; each with its date.

<details>
<summary>Hint — formatting a timestamp</summary>

`new Date(changedAt).toLocaleDateString()` formats for the user's
locale. In tests, that output varies by machine — assert on the
status text, or pass an explicit locale.
</details>

---

## Checkpoint

- ✅ HI-1 through HI-6 and UI-4 each have tests with their IDs
- ✅ In `psql`, `\d status_changes` shows `timestamptz`, the default,
  and `ON DELETE CASCADE`
- ✅ Manual check: add some applications on the **previous** migration,
  `alembic upgrade head`, and see their backfilled history in the UI
- ✅ CI green

Next: [06 — Pagination](06-pagination.md)
