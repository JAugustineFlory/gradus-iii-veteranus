# 08 — Debrief

**Requirements covered:** NF-1, NF-2, NF-3, and a final check of every
INF requirement.

**Outcome:** proof that the project meets its specification, a README a
stranger could follow, and a debrief across all three tiers.

---

## 1. Traceability audit (NF-1)

List every requirement ID mentioned in your tests. From the repo root
(Git Bash or macOS/Linux):

```bash
grep -rhoiE "(inf|co|ap|hi|ui)-?[0-9]+" \
  backend/tests frontend/src \
  | tr '[:lower:]' '[:upper:]' | tr -d '-' | sort -u
```

- **`grep -rhoiE`** — search recursively, print only the matching text,
  ignore case, use extended patterns.
- The two `tr` commands normalize `hi3`, `HI-3`, and `Hi-3` to `HI3`, so
  each ID appears once.

Compare the output with NF-1's list. Any ID missing is a requirement
you **claim** is met but haven't **proven**. Write the test.

---

## 2. Infrastructure checklist

These can't be unit tests; verify each by hand:

- [ ] **INF-1** `docker compose down` then `up -d` keeps your data
- [ ] **INF-2** Running the whole test suite leaves the development
      database unchanged
- [ ] **INF-4** Your test setup runs migrations; `create_all` appears
      nowhere in `backend/`
- [ ] **INF-6** `app/` uses only the async engine and `AsyncSession`
      with asyncpg (`grep -rw create_engine backend/app` finds
      nothing; only `create_async_engine` is used), and no request
      ever raises `MissingGreenlet`
- [ ] **INF-7** The badge shows **passing** on `main`
- [ ] **INF-8** Lowering coverage below a threshold fails CI
- [ ] **INF-9** Committing runs every local check

---

## 3. README (NF-2, NF-3)

A stranger with Docker, Node, and uv installed should get from `git
clone` to a running app and a green test suite using only your README.
Include:

- what the app is (one or two sentences) and the CI badge
- prerequisites
- first-time setup, in order (Docker, `.env` files, migrations, installs)
- how to run the app, and how to run the tests
- the **Decisions** section (NF-3)

**Test it:** do the fresh-clone check one last time, following *only*
your README. Every place you had to guess is a line missing from it.

---

## 4. After Action Review — across all three tiers

You've now built the same app three times, with less help each time.
Use that. Write your answers down.

1. **What was supposed to happen?** What did you expect Veteranus to
   demand that Tiro and Miles didn't?
2. **What actually happened?** Where did you spend the most time?
3. **Why the difference?** Look at your lesson 01 debrief from Miles.
   Did the same weak spots show up here? Did any disappear?
4. **Sustain and improve.** What habits will you carry into real
   projects? What will you do differently?

And one more:

5. **Compare your three `App.tsx` files** — Tiro's, Miles's, and this
   one. What changed, and why is each version better than the last?

---

## 5. Final self-check

Without looking, can you:

- [ ] Break a project into a WBS and a test list tagged with
      requirements
- [ ] Run PostgreSQL in Docker from memory, and explain images,
      containers, volumes, and port mappings
- [ ] Explain why Alembic's async `env.py` must be called from
      synchronous test code
- [ ] Configure an app with `pydantic-settings` and `.env` files
- [ ] Build a test database by running migrations, and isolate tests
      from each other
- [ ] Write a GitHub Actions workflow with a database service
- [ ] Enforce coverage thresholds
- [ ] Add a cascade delete that the database enforces, and prove it
- [ ] Write and **test** a data migration
- [ ] Paginate with limit/offset and return a total
- [ ] Write a generic Pydantic model and a generic TypeScript type
- [ ] Extract a custom hook and test it with `renderHook`
- [ ] Explain a fetch race condition and prevent it
- [ ] Derive state instead of duplicating it

---

## 6. From here to the internship

The internship's applicant tracker is this project's cousin. Map it:

| GRADUS | Applicant tracker |
| --- | --- |
| Companies | Positions or teams being hired for |
| Applications | Applicants |
| Status | Hiring stage (screening, interview, offer…) |
| Status history | Audit trail: who moved an applicant, and when |
| Pagination and filters | Browsing hundreds of applicants by stage |
| Data migration | Importing the old spreadsheet-and-email data |
| CI and coverage thresholds | How a team keeps shared code working |

### Where to go next

Each of these is a natural next layer; each has excellent official docs:

- **Authentication** — who is making the request? (FastAPI's security
  tutorial)
- **End-to-end tests** — a real browser clicking through the real app
  (Playwright)
- **Deployment** — running the app somewhere other than your laptop
- **Generated API clients** — creating `api.ts` automatically from
  FastAPI's OpenAPI schema

---

*Tiro* → *miles* → *veteranus.* Done. **Bene factum** — well done.
