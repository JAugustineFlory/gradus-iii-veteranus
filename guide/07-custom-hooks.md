# 07 — Custom hooks and race conditions

**Requirements covered:** UI-2, UI-5, UI-6

**Outcome:** `App.tsx` becomes a thin layout that uses custom hooks;
the hooks have their own tests; a loading indicator appears; and fast
clicking can never show stale data.

---

## Primer: custom hooks

A **custom hook** is a function whose name starts with `use` and which
calls other hooks (`useState`, `useEffect`, …). It lets you pull
**stateful logic** out of a component and reuse or test it on its own.

**Example from a different domain** — a toggle, used by any component
that shows and hides something:

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial)
  const toggle = () => setOn((current) => !current)
  return { on, toggle }
}

function Spoiler({ text }: { text: string }) {
  const { on, toggle } = useToggle()
  return (
    <>
      <button onClick={toggle}>{on ? 'Hide' : 'Show'}</button>
      {on && <p>{text}</p>}
    </>
  )
}
```

- The component doesn't know *how* the state works — it just uses it.
- Each component calling `useToggle` gets its **own** state.
- The **Rules of Hooks** apply inside custom hooks too.

For UI-6, `App` currently holds the data, the loading logic, and every
handler. Moving that into `useCompanies` and `useApplications` leaves
`App` doing one job: arranging components.

**Designing the hook's interface is your call.** Ask: what does `App`
need to *read* (items, total, loading, error…) and what does it need to
*do* (add, change status, delete, change page or filter…)? That's what
the hook returns.

Docs: <https://react.dev/learn/reusing-logic-with-custom-hooks>

---

## Primer: testing hooks with `renderHook`

You can't call a hook outside a component. `renderHook` renders a tiny
invisible component that calls it, and gives you its latest return value
as `result.current`:

```tsx
import { act, renderHook } from '@testing-library/react'

it('toggles', () => {
  const { result } = renderHook(() => useToggle())

  act(() => result.current.toggle())

  expect(result.current.on).toBe(true)
})
```

- **`result.current`** always holds the *latest* return value — re-read
  it after anything changes; don't store it in a variable.
- **`act(...)`** wraps anything that updates state, so React finishes
  re-rendering before you assert. For async actions:
  `await act(async () => { await result.current.add(...) })`.
- **`rerender(newProps)`** re-runs the hook with new arguments — how you
  test "the filter changed."
- For data that arrives later, `await waitFor(() => expect(...))` works
  just as it did with components.

Your hooks call the API module, so mock it with `vi.mock` exactly as in
Miles's `App` tests.

Docs:
<https://testing-library.com/docs/react-testing-library/api#renderhook>

---

## Primer: race conditions in data fetching (UI-5)

Picture this, with a slow network:

```text
t=0ms    filter → "interviewing"   request A sent
t=50ms   filter → "offer"          request B sent
t=300ms  B arrives → list shows offers                  ✓
t=900ms  A arrives → list shows interviewing… under "offer"  ✗
```

The responses arrived **out of order**, and the older one won. The
screen now says *Show: Offer* while listing interviews. This is a
**race condition**: the result depends on timing you don't control.

### The fix: effect cleanup

An effect can return a **cleanup function**. React runs it before the
effect runs again (and when the component goes away). That's exactly
the moment a request becomes stale.

The pattern from React's own docs, in general form:

```tsx
useEffect(() => {
  let ignore = false

  fetchSomething(input).then((result) => {
    if (!ignore) {
      setData(result)
    }
  })

  return () => {
    ignore = true
  }
}, [input])
```

When `input` changes, the cleanup marks the old request's closure as
`ignore = true`, so its late result is thrown away.

An alternative is **`AbortController`**, which also cancels the network
request itself. It needs the API module to accept a `signal`. Choose
either; record which.

Docs:
- <https://react.dev/learn/synchronizing-with-effects#fetching-data>
- <https://developer.mozilla.org/en-US/docs/Web/API/AbortController>

---

## Primer: controlling *when* a promise resolves, in tests

To test UI-5, the test has to make an **older** request finish **after**
a newer one. `mockResolvedValue` resolves immediately, so it can't.
Instead, hand out promises the test resolves by hand.

This small helper is test infrastructure, not app logic — use it as is:

```ts
function deferred<T>() {
  let resolve!: (value: T) => void
  const promise = new Promise<T>((r) => {
    resolve = r
  })
  return { promise, resolve }
}
```

A test for UI-5 then goes, in outline:

```text
first  = deferred(); second = deferred()
the mocked fetch returns first.promise on call 1, second.promise on call 2
render the hook (call 1 starts), then change its filter (call 2 starts)
resolve SECOND with the new results      ← the newer one lands first
resolve FIRST  with the old results      ← the older one lands late
assert: the hook still shows the second results
```

<details>
<summary>Hint — different results per call</summary>

Mocks have `.mockReturnValueOnce(...)`, which can be chained: the first
call gets the first value, the second call the second.
</details>

<details>
<summary>Hint — resolving inside act</summary>

Resolving causes state updates, so do it inside
`await act(async () => { second.resolve(...) })`.
</details>

**Prove the test works:** remove your `ignore` check, and the test must
fail. Put it back.

---

## Loading (UI-2) and derived state

"Loading" is true from the moment a request starts until it finishes or
fails. Storing it in its own `useState` and setting it in several places
is easy to get wrong — and newer ESLint rules object to setting state
directly in an effect's body.

<details>
<summary>Hint — derive it instead</summary>

Store *which request* the current data belongs to (for example the
filter and page it was loaded for). Loading is simply "the data we have
isn't for the filter and page we want." It's calculated during render,
never set by hand — so it can never be out of sync.
</details>

React's docs on this principle:
<https://react.dev/learn/choosing-the-state-structure#avoid-redundant-state>

---

## The payoff: refactoring under tests

Your `App` tests (as updated for pagination in lesson 06) mock the API
module and check what the user sees. They don't care whether the logic
lives in `App` or in a hook. So:

1. Write the hook tests (red), then the hooks (green).
2. Change `App` to use the hooks.
3. Run the `App` tests **unchanged**. If they pass, the refactor kept
   every behavior. If one fails, you found a regression before any user
   did.

That's the promise of TDD from Tiro lesson 00, paid out: **change code
without fear.**

---

## Checkpoint

- ✅ `App.tsx` imports nothing from `api.ts` (UI-6)
- ✅ `useCompanies` and `useApplications` each have their own test file
- ✅ The UI-5 test fails without the stale-response guard and passes
  with it
- ✅ UI-2 tests pass; you see **Loading…** briefly in the browser
  (throttle the network in DevTools → Network to see it clearly)
- ✅ Your existing `App` tests pass unchanged (or with changes you can
  justify under **Decisions**)
- ✅ Coverage thresholds met; CI green

Next: [08 — Debrief](08-debrief.md)
