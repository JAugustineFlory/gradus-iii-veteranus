# 01 — Requirements

This is the specification. Keep it open the whole time. Every
requirement has an **ID**; tag your tests with it (see lesson 00).

"**Must**" is required. "**Should**" is strongly recommended; skipping it
is a decision you record in your README's **Decisions** section.

---

## INF — Infrastructure

| ID | Requirement |
| --- | --- |
| **INF-1** | PostgreSQL **must** run in Docker, started with `docker compose up -d` from the repo root. Data **must** survive `docker compose down` followed by `up` (a named volume). |
| **INF-2** | There **must** be two databases: one for development and one for tests. Tests **must never** touch the development database. |
| **INF-3** | Backend configuration **must** come from environment variables through `pydantic-settings`, and **must** also be readable from a `backend/.env` file. A `backend/.env.example` listing every setting **must** be committed; `.env` **must not** be. Settings: `DATABASE_URL`, `TEST_DATABASE_URL`, `FRONTEND_ORIGIN`. |
| **INF-4** | The test database's schema **must** be built by running your Alembic migrations — not `Base.metadata.create_all`. |
| **INF-5** | Every backend test **must** start with empty tables, and tests **must** pass in any order. No test may assume a particular `id` value. |
| **INF-6** | No SQLite-specific code may run against PostgreSQL. |
| **INF-7** | A GitHub Actions workflow **must** run on every push and pull request: backend lint and tests (against a PostgreSQL service) and frontend lint, type-check, and tests. The README **must** show the workflow's status badge. |
| **INF-8** | Coverage thresholds **must** be enforced, failing the run when not met: **95%** backend, **90%** frontend (lines). |
| **INF-9** | The Husky pre-commit hook **must** run the same checks as in Miles. |

---

## CO — Companies

| ID | Requirement |
| --- | --- |
| **CO-1** | `POST /companies` with `{"name": …}` creates a company and returns `201` with `{"id", "name"}`. Surrounding whitespace is stripped; the name must be 1–100 characters after stripping (`422` otherwise). |
| **CO-2** | `GET /companies` returns every company, sorted by name. |
| **CO-3** | Company names are unique **case-insensitively**: creating `"acme"` when `"Acme"` exists returns `409` with detail `"Company already exists"`. |

---

## AP — Applications

| ID | Requirement |
| --- | --- |
| **AP-1** | `POST /applications` with `{company_id, role, status?, applied_on}` returns `201`. `role`: stripped, 1–100 characters. `status`: one of `applied`, `interviewing`, `offer`, `rejected`; default `applied`. An unknown `company_id` returns `422` with detail `"Company not found"`. |
| **AP-2** | Every application in a response has the shape `{id, role, status, applied_on, company: {id, name}}`. |
| **AP-3** | `GET /applications/{id}` returns one application, or `404` with detail `"Application not found"`. |
| **AP-4** | `GET /applications` returns a **page**: `{"items": [...], "total": N, "limit": L, "offset": O}`. Items are ordered by `id`. `limit` defaults to 20 and must be 1–100; `offset` defaults to 0 and must be ≥ 0 (`422` otherwise). `total` counts **every** matching application, not just those on the page. |
| **AP-5** | `GET /applications` accepts optional `status` and `company_id` filters, alone or together. Filters apply to both `items` and `total`. An unknown `status` returns `422`. |
| **AP-6** | `PATCH /applications/{id}` changes only the fields sent. Unknown application → `404`; unknown `company_id` → `422` `"Company not found"`; invalid values → `422`. |
| **AP-7** | `DELETE /applications/{id}` returns `204`, or `404` if missing. |

---

## HI — Status history

| ID | Requirement |
| --- | --- |
| **HI-1** | A `status_changes` table records status changes: `id`, `application_id` (foreign key), `from_status` (nullable), `to_status`, and `changed_at` (a time-zone-aware timestamp **set by the database**). |
| **HI-2** | Creating an application records **one** change: `from_status` null → the initial status. |
| **HI-3** | A `PATCH` that **changes** the status records **one** change from the old status to the new. A `PATCH` that sets the same status, or doesn't include status, records **nothing**. |
| **HI-4** | `GET /applications/{id}/history` returns that application's changes, **oldest first**, each as `{from_status, to_status, changed_at}`. Missing application → `404`. |
| **HI-5** | Deleting an application deletes its history, **enforced by the database** (not only by application code). |
| **HI-6** | The migration that creates `status_changes` **must backfill** it: every application that already exists gets one row, `from_status` null → its current status, with `changed_at` equal to its `applied_on` date at midnight UTC. A test **must** prove this by running the migrations against data inserted before them. |

---

## UI — Frontend

| ID | Requirement |
| --- | --- |
| **UI-1** | Everything the Miles UI did still works: adding companies, adding applications, changing status, deleting, filtering by status, and showing a `role="alert"` error for any failed action. |
| **UI-2** | While applications are loading, the page shows **Loading…** in an element with `role="status"`. It disappears when loading finishes or fails. |
| **UI-3** | Applications are shown 20 per page with **Previous** and **Next** buttons and the text **Showing X–Y of Z** (e.g. *Showing 21–40 of 57*). Previous is disabled on the first page; Next is disabled on the last. Changing the status filter returns to the first page. With no applications, the "Showing" text is hidden. |
| **UI-4** | Each application has a **History** button (accessible name `History for {role} at {company}`) that shows and hides its history. The first entry reads **Created as {Status}**; later ones read **{From} → {To}**, each with its date. |
| **UI-5** | If the filter or page changes while an earlier request is still in flight, only the **latest** request's result may be shown — a slow, older response must never overwrite a newer one. |
| **UI-6** | `App.tsx` **must not** call the API module directly. Data loading and changes live in **custom hooks** (at least `useCompanies` and `useApplications`), each with its own tests. |

---

## NF — Non-functional

| ID | Requirement |
| --- | --- |
| **NF-1** | Every **CO**, **AP**, **HI**, and **UI** requirement, plus **INF-3** and **INF-5**, appears in the name of at least one test. (The other INF requirements are infrastructure; lesson 08 verifies them with a checklist.) |
| **NF-2** | The README explains how to run the app and the tests from a fresh clone, including Docker. |
| **NF-3** | The README has a **Decisions** section recording each place you interpreted an ambiguous requirement or skipped a "should." |

---

## Clarifications you'll probably want

<details>
<summary>Does <code>DELETE</code> of a company exist?</summary>

No. Companies can't be deleted in Veteranus. (What *should* happen to a
company's applications if one were deleted is a good design question to
think through, though.)
</details>

<details>
<summary>Does <code>PATCH</code> of <code>status</code> on a filtered page remove the item from view?</summary>

Unspecified. Decide, and record the decision.
</details>

<details>
<summary>What if <code>offset</code> is past the end?</summary>

Return `200` with empty `items` and the correct `total`. That's not an
error: the collection simply has nothing there.
</details>
