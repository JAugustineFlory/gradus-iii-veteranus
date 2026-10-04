# GRADUS III — Veteranus

> **🚧 Beta.** This guide has not yet been fully tested end to end by
> students. Tool versions change, and some commands or details may not
> match what you see. Check
> [`guide/troubleshooting.md`](guide/troubleshooting.md) first, then
> [report it](#found-a-problem).

**GRADUS**: **G**uided **R**epetitions in **A**pplied **D**evelopment
**U**sing the **S**tack

*Veteranus* (Latin): a veteran — from *vetus*, "old," "experienced."
The last of three tiers.

| Tier | Latin | Guidance level |
| --- | --- | --- |
| I | *tiro* | Full worked examples. Every line explained. |
| II | *miles* | Tests given to you. You write the code. |
| **III** | ***veteranus*** | **Requirements only. You write the tests and the code.** |

---

## What you'll build

The job application tracker, one last time, from an empty folder — now
built the way a production team would:

- The same stack as Tiro and Miles — FastAPI, Pydantic, async
  SQLAlchemy with asyncpg, Alembic, and PostgreSQL in Docker (host port
  **5435**) — set up entirely from memory
- **Typed settings** with pydantic-settings and `.env` files
- Backend tests running against a PostgreSQL test database built by
  **your own migrations**, not `create_all`
- **Continuous integration**: every push to GitHub runs every check in
  the cloud
- A **status history** for each application, recording every change —
  plus a **data migration** that backfills history for applications that
  already exist
- **Pagination** for long lists
- **Custom React hooks** that move data logic out of `App`, with
  protection against out-of-order network responses
- **Enforced coverage thresholds**

## Prerequisites

Finish **GRADUS I — Tiro** and **GRADUS II — Miles** first. Veteranus
names concepts from them without re-explaining. Your Miles code is your
most useful reference.

---

## How Veteranus works

There are **no provided tests** and **no step-by-step instructions**.
You get:

1. **A written specification** —
   [`guide/01-requirements.md`](guide/01-requirements.md). Every
   requirement has an ID (like `HI-3`) and acceptance criteria precise
   enough to turn into tests.
2. **Primers** for each new concept: what it is, why it exists, a small
   example *from a different domain*, and links to the official docs.
   You translate the idea to this app yourself.
3. **Hints**, concept-level only. They point you at the right tool or
   doc section — they don't hand you the code.
4. **A plan you write.** Before coding, you break the work down and list
   your tests (lesson 00).

**No AI assistant required.** The requirements define "done." Your tests
prove it. The primers and docs cover every new concept.

---

## Skills

### Reinforced (no guidance)

Everything from Tiro and Miles: uv, Docker Compose, PostgreSQL and
`psql`, FastAPI, routers, Pydantic validation, `async`/`await`, async
SQLAlchemy with asyncpg, relationships and eager loading, async pytest
fixtures, Alembic (async template), Vite, React, TypeScript, Vitest,
Testing Library, mocking `fetch` and modules, error handling, Husky,
Ruff, ESLint.

### New in Veteranus

**Planning**

- Breaking a project into a **work breakdown structure** (WBS)
- Writing a **test list** from requirements before writing tests
- **Requirement traceability** — linking each test to the requirement
  it proves

**Infrastructure**

- **`pydantic-settings`** for typed configuration and `.env` files
- Running **Alembic programmatically** from a test fixture, and how
  that interacts with the async migration template
- **GitHub Actions**: workflows, jobs, steps, service containers
- **Coverage thresholds** that fail the build

**Backend**

- Test databases built by **running your migrations**, not `create_all`
- Isolating tests on a shared database
- **Cascade deletes**
- **Database-generated timestamps** and time zones
- **Data migrations** — changing existing rows safely
- **Testing a migration**
- **Pagination** with limit/offset and a response envelope
- **Generic Pydantic models**
- **Case-insensitive** uniqueness checks

**Frontend**

- **Custom hooks** and testing them with `renderHook`
- **Race conditions** in data fetching, and effect cleanup
- **Controlled promises** in tests
- **Derived state** instead of duplicated state
- Loading indicators and a pager

---

## Getting started

**Don't work directly in this repo.** On its GitHub page, click **Use
this template → Create a new repository**, then clone *your* copy.

| # | Lesson | Outcome |
| --- | --- | --- |
| 00 | [Operation order](guide/00-operation-order.md) | Your WBS and test list |
| 01 | [Requirements](guide/01-requirements.md) | *(the spec — keep it open)* |
| 02 | [Settings and the test database](guide/02-docker-and-postgres.md) | Docker from memory, typed settings, a migration-built test database |
| 03 | [Continuous integration](guide/03-continuous-integration.md) | Every push checked on GitHub |
| 04 | [Rebuild the core](guide/04-rebuild-the-core.md) | Miles's features on PostgreSQL |
| 05 | [Status history and data migration](guide/05-status-history.md) | An audit trail, backfilled and tested |
| 06 | [Pagination](guide/06-pagination.md) | Paged API and UI |
| 07 | [Custom hooks and race conditions](guide/07-custom-hooks.md) | A thin `App`, correct under fast clicking |
| 08 | [Debrief](guide/08-debrief.md) | AAR, final review, what's next |

Also: [`guide/troubleshooting.md`](guide/troubleshooting.md) and
[`guide/glossary.md`](guide/glossary.md).

**Time:** plan on 16–24 hours.

---

## Why it's built this way

Tiro showed complete solutions. Miles removed the solutions but kept the
tests. Veteranus removes the tests too — the last step of **fading**,
where you move from studying examples to solving problems independently
(Renkl and Atkinson). Help that's still there at this stage would get in
the way (Kalyuga's *expertise reversal effect*).

The primers use examples from **other domains** on purpose. Translating
an idea from one context to another is what builds **transfer** — the
ability to use a skill somewhere new, which is the whole point.

---

## Found a problem?

On this repo's GitHub page, open **Issues → New issue** with the lesson,
what you expected, what happened (with the full error), and your
operating system.
