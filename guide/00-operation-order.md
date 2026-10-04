# 00 — Operation order

In Tiro and Miles, the guide planned the work for you. Now you plan it.
This lesson gives you the briefing and two planning tools. **Don't
write code until both are done.**

---

## The briefing (SMEAC)

**Situation.** Your Miles app works, but it wouldn't survive contact
with real users: its settings are untyped `os.getenv` calls, its tests
never exercise the migrations, nobody checks it automatically, it forgets the history of each application, it would
choke on a thousand applications, and `App.tsx` is doing too much.

**Mission.** Rebuild the tracker from an empty folder so that every
requirement in [`01-requirements.md`](01-requirements.md) is met and
proven by a test you wrote.

**Execution.** The recommended order — each phase builds on the last:

1. Infrastructure: Docker and PostgreSQL from memory, typed settings, a
   migration-built test database (lesson 02)
2. Continuous integration, *before* features (lesson 03)
3. Core features from Miles, with tests you wrote (lesson 04)
4. Status history and the data migration (lesson 05)
5. Pagination (lesson 06)
6. Custom hooks and race conditions (lesson 07)

**Admin.** Your references, in order: the requirements, the lesson
primers, the official docs, your Miles code, your Tiro code.

**Command.** You are in charge. When the requirements are ambiguous,
make a decision, write it down in your repo's README under a
**Decisions** heading, and move on. Real specifications always have
gaps; recording how you filled them is part of the job.

---

## Tool 1 — Work breakdown structure (WBS)

A **WBS** breaks a large goal into smaller and smaller pieces until each
piece is small enough to finish in one sitting (roughly 30–90 minutes).
It answers "what exactly do I have to do?" before you start doing it.

**Example from a different domain** — hosting a dinner:

```text
1. Dinner party
   1.1 Guests
       1.1.1 Choose guest list
       1.1.2 Send invitations
       1.1.3 Collect dietary needs
   1.2 Food
       1.2.1 Plan menu (depends on 1.1.3)
       1.2.2 Shop
       1.2.3 Cook
   1.3 Venue
       1.3.1 Clean
       1.3.2 Set table
```

Notice:

- Each level is a **breakdown** of the level above, not a sequence.
- The leaves (lowest items) are concrete tasks you'd know when you'd
  finished.
- **Dependencies** are noted where order matters.

### Your task

Create `PLAN.md` at the root of your repo. Write a WBS for Veteranus
with the six phases above as the top level. Break each phase down using
the requirements. Aim for 30–60 leaves in total.

<details>
<summary>Hint — how deep to go</summary>

"Pagination" is too big. "Backend pagination" is still too big.
"Return `total` alongside items" is about right — one or two tests and a
few lines of code.
</details>

---

## Tool 2 — The test list

Kent Beck's TDD practice starts every session with a **test list**: a
plain list of every test you expect to write, before writing any. You
add to it as you discover cases, and cross items off as they go green.
It keeps you from forgetting edge cases and from trying to do
everything at once.

**Example from a different domain** — a function that splits a
restaurant bill:

```text
[ ] splits evenly between 2 people
[ ] splits evenly between 3 people with a remainder — who pays the extra cent?
[ ] adds the tip before splitting
[ ] rejects 0 people
[ ] rejects a negative total
[ ] handles a total of 0
```

Each line is a behavior, phrased so it could be a test name.

### Your task

Add a **Test list** section to `PLAN.md`. Go through
`01-requirements.md` one requirement at a time and list every test that
requirement needs. **Put the requirement ID at the start of each line:**

```text
[ ] AP-4 list rejects a limit above 100
[ ] HI-3 changing status records one change from old to new
[ ] HI-3 patching without changing status records nothing
```

### Requirement traceability

Carry the ID into the tests themselves, so anyone can find which test
proves which requirement:

```python
async def test_hi3_patch_without_status_change_records_nothing(client):
    ...
```

```ts
it('UI-5 ignores a response that arrives after a newer request', ...)
```

When the project is done, every testable requirement ID should appear in
at least one test name (NF-1 lists which). That's **traceability**:
proof, not just a claim, that each requirement is met.

Pick one ID format per language and stick to it — for example
`test_hi3_…` in Python and `'HI-3 …'` in TypeScript — so a search can
find them all.

---

## Rules of engagement

- **Red first, always.** No tests are provided, so it's on you. Watch
  every new test fail before making it pass.
- **Commit on every green**, and push often — CI (lesson 03) checks
  every push.
- **Never leave CI red** on `main`. If it breaks, fixing it comes before
  anything else.
- **Hints are concept-level.** They point to the right idea or doc.
  If a hint isn't enough, the docs are the next step.

---

## Checkpoint

- ✅ `PLAN.md` exists with a WBS (six phases, 30+ leaves) and a test
  list tagged with requirement IDs
- ✅ You've read `01-requirements.md` end to end at least once

Commit `PLAN.md` — it's the first file in your repo.

Next: [01 — Requirements](01-requirements.md)
