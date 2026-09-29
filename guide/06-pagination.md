# 06 — Pagination

**Requirements covered:** AP-4, AP-5 (with pages), UI-3

**Outcome:** the API returns applications a page at a time with a total
count, and the UI pages through them.

---

## Primer: why paginate

Returning *every* row works with ten applications. With ten thousand,
every request moves megabytes, the database does needless work, and the
browser renders a list nobody can scroll. **Pagination** returns one
slice at a time.

### Limit and offset

The simplest scheme: **`limit`** = how many to return, **`offset`** =
how many to skip first.

```text
rows:     1  2  3  4  5  6  7  8  9  10 11 12
limit=5, offset=0   → 1–5
limit=5, offset=5   → 6–10
limit=5, offset=10  → 11–12
```

In SQL that's `ORDER BY id LIMIT 5 OFFSET 5`. **Order is essential**:
without `ORDER BY`, the database may return rows in any order, and
pages could overlap or skip rows.

SQLAlchemy: `.order_by(...).limit(n).offset(m)`.

> **The other scheme.** Large systems often use **keyset** (or
> "cursor") pagination instead — "the 20 rows after id 480" — because a
> big `OFFSET` gets slower the further you page, and rows inserted
> while you page can shift what you see. Limit/offset is the right
> choice here. Know the other exists.

### The response envelope

A bare list can't say how many rows exist in total, so the UI couldn't
show "of 57." AP-4 wraps the list in an **envelope**:

```json
{"items": [...], "total": 57, "limit": 20, "offset": 20}
```

`total` needs a second query: the same filters, but **counting** instead
of fetching, and **without** limit and offset.

<details>
<summary>Hint — counting a filtered query</summary>

Build the filtered query once. Count it with
`select(func.count()).select_from(filtered.subquery())`, then separately
add order, limit, and offset to fetch the page.
</details>

Docs:
<https://docs.sqlalchemy.org/en/20/tutorial/data_select.html#ordering-grouping-limit-offset>

---

## Primer: validated query parameters

Miles's `status` query parameter was validated by its type. Numbers with
**ranges** use FastAPI's `Query` inside `Annotated`:

**Example from a different domain** — a search endpoint with a minimum
query length:

```python
@router.get("/search")
def search(
    q: Annotated[str, Query(min_length=3)],
    ...
```

For numbers, look up `ge` ("greater than or equal") and `le` ("less
than or equal").

Docs:
<https://fastapi.tiangolo.com/tutorial/query-params-str-validations/>

---

## Primer: generic Pydantic models

You'll want the same envelope for any list someday — companies,
history. A **generic** model takes the item type as a parameter, just
like `request<T>` did in TypeScript:

**Example from a different domain:**

```python
from typing import Generic, TypeVar

from pydantic import BaseModel

T = TypeVar("T")


class Envelope(BaseModel, Generic[T]):
    data: T
    warnings: list[str] = []


# Envelope[Book] is an envelope whose data is a Book
```

`response_model=Page[ApplicationRead]` then tells FastAPI exactly what
the response looks like, and `/docs` shows it.

Docs: <https://docs.pydantic.dev/latest/concepts/models/#generic-models>

---

## Changing an API that's already in use

Your frontend expects `GET /applications` to return an array. The moment
the backend returns an envelope, the frontend breaks. This is a
**breaking change**.

Here, you control both sides and deploy them together, so the fix is to
change both in the same piece of work — and let the tests on each side
prove it:

1. Change the backend tests for AP-4 → red → make them green.
2. The frontend's **API module tests** still assume an array. Update
   them for the envelope → red → green.
3. Update the types and anything that consumes them.

When other teams (or a mobile app you can't update instantly) depend on
an API, you can't do this. Teams then **version** their APIs
(`/v2/applications`) and keep the old version running for a while. Worth
knowing; not needed here.

---

## Frontend (UI-3)

### Connect the dots

- The page number, the page size (20), and the filter together decide
  which request to make. What's the offset for page 3?
- "Showing **X–Y** of **Z**" — X, Y, and Z can all be **calculated**
  from the offset, the number of items returned, and the total. Don't
  store them separately.
- "Changing the filter returns to the first page" — does that belong in
  an effect, or in the handler that changes the filter?

<details>
<summary>Hint — the math</summary>

With `offset` 20 and 17 items returned: X = offset + 1 = 21,
Y = offset + items.length = 37. Next is disabled when
`offset + items.length >= total`.
</details>

<details>
<summary>Hint — resetting the page</summary>

Set both the filter *and* the page in the filter's change handler. An
effect that "watches the filter and resets the page" would first fetch
the wrong page, then the right one.
</details>

### A generic page type

The TypeScript twin of `Page[T]`:

```ts
export type Page<T> = {
  items: T[]
  total: number
  limit: number
  offset: number
}
```

---

## Checkpoint

- ✅ AP-4 and UI-3 tests pass, including the edges: `limit` 0 and 101,
  negative `offset`, `offset` past the end, `total` with filters
- ✅ Test data with **more than one page** exists in at least one test
  on each side
- ✅ In the browser: create 21+ applications and page through them
- ✅ CI green

Next: [07 — Custom hooks and race conditions](07-custom-hooks.md)
