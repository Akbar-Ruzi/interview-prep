# Interview Notes: TypeScript, React, Next.js

> Every snippet is from my project (Talent Pipeline Tracker, Next.js 16.4, React 19.3, TypeScript 5).
> For each concept: what it is, the project code, and a likely interview question.

## Contents

- [TypeScript](#typescript)
  1. Union literal types
  2. Utility types: `Pick`, `Omit`, `Partial`, `Record`
  3. `unknown` vs `any`
  4. Type narrowing
  5. Type guards (`value is T`)
  6. Runtime validation
  7. Generics
  8. Type assertions (`as`)
  9. Intersection types (`&`)
  10. Optional chaining and nullish coalescing
  11. Type-only imports
- [JavaScript fundamentals](#javascript-fundamentals)
  12. Short-circuit evaluation
  13. `async`/`await` with `try`/`catch`/`finally`
  14. `fetch` does not reject on HTTP errors
  15. Closures
  16. Destructuring, spread, and rest
  17. `URLSearchParams` and `encodeURIComponent`
- [React](#react)
  18. Custom hooks
  19. `useState` and functional updates
  20. `useEffect`, cleanup, and dependency arrays
  21. Race conditions in data fetching (the `ignore` flag)
  22. `useRef` vs `useState`
  23. Adjusting state during render (instead of an effect)
  24. Derived values instead of state
  25. `useCallback`
  26. `useEffectEvent`
  27. Debouncing
  28. Conditional rendering
  29. Lists and `key`
  30. Reusable components and default props
  31. Loading, error, and empty states
  32. Strict Mode runs effects twice in development
  33. IntersectionObserver (infinite scroll)
  34. Accessibility in components
- [Next.js (App Router)](#nextjs-app-router)
  35. File-based routing and dynamic segments
  36. Server Components vs Client Components
  37. `useSearchParams`, `useRouter`, `usePathname`
  38. `router.push` vs `router.replace`
  39. The URL as state
  40. `Suspense` boundaries
  41. `params` and `searchParams` are Promises
  42. `PageProps` and `LayoutProps` typed helpers
  43. `Link` and client-side navigation
  44. `cacheComponents` and `<Activity>`
  45. Metadata API and `next/font`
  46. Environment variables (`NEXT_PUBLIC_`)
- [Promises and API calls](#promises-and-api-calls)
  47. Promises and `Promise<T>`
  48. An `async` function always returns a Promise
  49. Generic syntax: `<T>`, not `(T)`
  50. HTTP methods: GET, POST, PUT, PATCH, DELETE
  51. A typed API layer (one function per endpoint)
- [Timers and save feedback](#timers-and-save-feedback)
  52. Timers in effects: auto-dismiss vs debounce vs throttle
  53. Effect cleanup for timers
  54. Save feedback: saving, success, error
- [Forms and validation](#forms-and-validation)
  55. `keyof`, `Record`, and `Partial` together: a typed errors object
- [Sharing state across components](#sharing-state-across-components)
  56. The Context API (`createContext`, provider, `useContext`)
  57. `useMemo` and `useCallback` for a context value
  58. Signal vs data: refetch after a save instead of patching
  59. A type-safe generic setter: `K extends keyof T`, `T[K]`, `NonNullable`

---

## TypeScript

### 1. Union literal types

A type that allows only a fixed set of exact values. Typos are caught at compile time, and editors autocomplete the options.

```ts
// types/candidate.ts
export type CandidateStatus =
  | "received"
  | "in_progress"
  | "selected"
  | "discarded";
```

**Interview Q:** *Union literal type vs `enum`?* A union has no runtime code and is just strings. An `enum` generates a JavaScript object. Many teams prefer unions plus a `const` object for labels (see `STATUS_LABELS`).

### 2. Utility types: `Pick`, `Omit`, `Partial`, `Record`

Built-in types that build new types from existing ones, so there is one source of truth.

```ts
// types/candidate.ts
// PUT body = Candidate minus server-owned fields
export type UpdateCandidatePayload = Omit<
  Candidate,
  "id" | "status" | "stage" | "applied_at" | "updated_at" | "notes_count"
>;

// PATCH body = only status/stage, both optional
export type UpdateCandidateStatusPayload = Partial<
  Pick<Candidate, "status" | "stage">
>;

// Filters = a subset of the query params
export type ListFilters = Pick<GetRecordsParams, "status" | "stage" | "search">;
```

```ts
// lib/constants.ts: Record forces a label for EVERY status
export const STATUS_LABELS: Record<CandidateStatus, string> = {
  received: "Received",
  in_progress: "In progress",
  selected: "Selected",
  discarded: "Discarded",
};
```

- `Pick<T, K>`: keep only keys `K`.
- `Omit<T, K>`: drop keys `K`.
- `Partial<T>`: make every key optional.
- `Record<K, V>`: an object with keys `K` and values `V`. If you add a new status to the union, TypeScript errors until you add its label.

**Interview Q:** *Why derive `UpdateCandidatePayload` from `Candidate` instead of writing it out?* If `Candidate` changes, the payload updates automatically, so the two types can't drift apart.

### 3. `unknown` vs `any`

- `any` turns type checking off. You can do anything with it, which is unsafe.
- `unknown` means "could be anything, prove it first." You must narrow it before using it, which is safe.

Use `unknown` for data from outside the program (API responses, `JSON.parse`, user input).

```ts
// lib/api/records.ts: the response starts as unknown
const data = await apiFetch<unknown>(`/records/${encodeURIComponent(id)}`, {
  method: "PATCH",
  body: JSON.stringify(payload),
});
if (!isCandidate(data)) {
  throw new Error("The updated candidate came back in an unexpected format.");
}
return data; // now typed as Candidate
```

**Interview Q:** *When would you use `unknown`?* When you don't control the data's shape: network responses, `catch (err)` values, and parsed JSON.

### 4. Type narrowing

TypeScript refines a broad type to a specific one inside a branch, based on a runtime check. Common narrowing checks:

| Check | Narrows to |
|---|---|
| `typeof x === "string"` | `string` |
| `x !== null` | removes `null` |
| `x instanceof Error` | `Error` |
| `Array.isArray(x)` | `any[]` |
| A custom type guard | whatever it declares |

```ts
// hooks/useCandidateFeed.ts: catch gives `unknown`; instanceof narrows it
} catch (err) {
  setError(err instanceof Error ? err.message : "Could not load candidates.");
}
```

```ts
// lib/utils/candidate-urls.ts: instanceof picks the right branch
const entries =
  searchParams instanceof URLSearchParams
    ? searchParams.entries()
    : Object.entries(searchParams);
```

**Interview Q:** *Why can't you write `err.message` directly in a `catch`?* `err` is `unknown`. Anything can be thrown, including a string, so you narrow it first.

### 5. Type guards (`value is T`)

A function that returns `boolean` and has the return type `value is T`. When it returns `true`, TypeScript treats the argument as `T` in that branch. It's a way to write your own narrowing.

```ts
// lib/api/validators/common-guards.ts
export function isObject(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}

export function isStringOrNull(value: unknown): value is string | null {
  return typeof value === "string" || value === null;
}
```

```ts
// lib/api/validators/candidate-guards.ts
export function isCandidate(item: unknown): item is Candidate {
  return (
    isObject(item) &&                     // after this, item.id etc. can be read
    typeof item.id === "string" &&
    typeof item.full_name === "string" &&
    isStringOrNull(item.linkedin_url) &&
    Object.hasOwn(STATUS_LABELS, item.status) &&
    // ...
    typeof item.notes_count === "number"
  );
}
```

**The key difference:** a plain `boolean` return type tells TypeScript nothing. The project hit this directly: `isListCandidate(item): boolean` had to become `isCandidate(item): item is Candidate` before `return data` would type-check as `Promise<Candidate>`.

**Interview Q:** *What's the risk of a type guard?* TypeScript trusts it completely. If the checks are incomplete, it can claim something is a `Candidate` when it isn't, so it has to check every field you rely on.

### 6. Runtime validation

TypeScript types are erased when the code is compiled. They don't exist in the browser. A type like `Candidate` can't stop a malformed API response, so you check the data at runtime at the boundary, where it enters the app.

```ts
// lib/api/records.ts
const data = await apiFetch<unknown>(qs ? `/records?${qs}` : "/records");
if (!isCandidateListResponse(data)) {
  throw new Error("The candidate list came back in an unexpected format.");
}
return data;
```

Result: a broken API answer shows an error message instead of a blank list or a crash deep in the UI.

**Interview Q:** *Doesn't TypeScript already guarantee the response type?* No. `fetch().json()` returns `any`. TypeScript only checks your code, not the data. Libraries like **Zod** do the same job declaratively (`schema.parse(data)`).

### 7. Generics

Type parameters, so one function works with many types while staying type-safe.

```ts
// lib/api/api-fetch.ts
export async function apiFetch<T>(
  path: string,
  options: ApiFetchOptions = {},
): Promise<T> {
  // ...
  return JSON.parse(text) as T;
}

// caller chooses T
const data = await apiFetch<unknown>("/records");
```

```ts
// Generic built-in types
useState<Candidate[]>([]);
useState<string | null>(null);
useRef<HTMLDivElement>(null);
```

**Interview Q:** *Why call `apiFetch<unknown>` rather than `apiFetch<Candidate>`?* `apiFetch<Candidate>` would be a lie: nothing has checked the data yet. `unknown` forces the type guard to run first.

### 8. Type assertions (`as`)

`as` tells the compiler "trust me." It doesn't check anything at runtime. Use it only right after a real check.

```ts
// hooks/useRecordsFilters.ts: assertion is safe because hasOwn checked it first
function toStatus(value: string | null): CandidateStatus | undefined {
  return value && Object.hasOwn(STATUS_LABELS, value)
    ? (value as CandidateStatus)
    : undefined;
}
```

```ts
// lib/api/api-fetch.ts: unchecked assertion; that's why callers pass <unknown>
return JSON.parse(text) as T;
```

**Interview Q:** *Type guard vs assertion?* A guard proves the type with a runtime check. An assertion only claims it.

### 9. Intersection types (`&`)

Combines types: the result has all the properties of each.

```ts
// lib/api/api-fetch.ts: every fetch option, plus one of our own
type ApiFetchOptions = RequestInit & {
  allowEmpty?: boolean;
};

// hooks/useCandidateTable.ts
type CandidateTableOptions = ListFilters & {
  page: number;
  onPageOutOfRange: (page: number) => void;
};
```

**Interview Q:** *`type` with `&` vs `interface` with `extends`?* They're mostly interchangeable for objects. `interface` can be reopened (declaration merging). `type` can also express unions, tuples, and mapped types.

### 10. Optional chaining and nullish coalescing

- `a?.b` returns `undefined` instead of throwing when `a` is `null` or `undefined`.
- `a ?? b` uses `b` only when `a` is `null` or `undefined`. `||` would also replace `0` and `""`.

```ts
// lib/api/records.ts
if (params.search?.trim()) query.set("search", params.search.trim());

// hooks/useRecordsFilters.ts
const page = pageParam ?? 1;

// hooks/useCandidateFeed.ts
const filterKey = `${status ?? ""}|${stage ?? ""}|${search ?? ""}`;
```

**Interview Q:** *`??` vs `||`?* `0 || 5` is `5`, but `0 ?? 5` is `0`. Use `??` when `0`, `""`, or `false` are valid values.

### 11. Type-only imports

`import type` is erased at compile time, so no runtime code is bundled for it.

```ts
import type { Candidate, CandidateListResponse } from "@/types/candidate";
```

---

## JavaScript fundamentals

### 12. Short-circuit evaluation

`&&` stops at the first falsy value, and `||` stops at the first truthy value. The rest isn't evaluated.

This is what makes chained type guards safe: `item.id` is only read after `isObject(item)` has passed.

```ts
// lib/api/validators/candidate-guards.ts
return (
  isObject(item) &&              // if false, stop: item.id is never read
  typeof item.id === "string" &&
  // ...
);
```

```ts
// hooks/useCandidateFeed.ts: guard clause with ||
if (loadingMoreRef.current || ended || candidates.length >= total) return;
```

```tsx
// components/shared/error-state.tsx: render only if provided
{onRetry && <button onClick={onRetry}>Retry</button>}
```

**Interview Q:** *What's the pitfall of `{count && <List />}` in JSX?* When `count` is `0`, React renders the number `0`. Use `count > 0 &&` or a ternary instead.

### 13. `async`/`await` with `try`/`catch`/`finally`

- `finally` runs whether the request succeeds or fails, which makes it the right place to turn off loading.

```ts
// hooks/useCandidateTable.ts
async function load() {
  try {
    const response = await getRecords({ page, limit: PAGE_SIZE, status, stage, search });
    if (!ignore) { setResult(response); setError(null); }
  } catch (err) {
    if (!ignore) setError(err instanceof Error ? err.message : "Could not load candidates.");
  } finally {
    if (!ignore) setLoading(false);
  }
}
```

### 14. `fetch` does not reject on HTTP errors

`fetch` only rejects on network failure. A 404 or 500 still resolves, so you have to check `response.ok`.

```ts
// lib/api/api-fetch.ts
let response: Response;
try {
  response = await fetch(buildUrl(path), { ...init, headers });
} catch {
  throw new Error("Could not reach the server. Check your connection and try again.");
}

if (!response.ok) {
  throw new Error(`Request failed (${response.status} ${response.statusText}).`);
}
```

The helper also reads `response.text()` before parsing, which lets it handle empty bodies (DELETE) and invalid JSON separately.

**Interview Q:** *Does `fetch` throw on a 500?* No. That's the most common fetch interview trap.

### 15. Closures

A function remembers the variables from the scope where it was created.

```ts
// hooks/useCandidateFeed.ts: isOutdated "remembers" listId from when loadMore started
const listId = listIdRef.current;
const isOutdated = () => listId !== listIdRef.current;
```

The cleanup function in an effect is also a closure. It captures that run's `ignore` or `timer`.

**Interview Q:** *What is a stale closure in React?* A callback that captured an old render's state. Fixes include functional updates, refs, and `useEffectEvent` (see #26).

### 16. Destructuring, spread, and rest

```ts
// lib/api/api-fetch.ts: pull one option out, keep the rest
const { allowEmpty = false, ...init } = options;

// components/candidate-list/candidate-list.tsx: rest in object destructuring
const { loadMoreAreaRef, ...feed } = useCandidateFeed(filters);

// spread to append without mutating
setCandidates((prev) => [...prev, ...response.data]);
```

### 17. `URLSearchParams` and `encodeURIComponent`

```ts
// lib/api/records.ts: building a query string safely
const query = new URLSearchParams();
if (params.page) query.set("page", String(params.page));

// encode a path segment so "/" or "?" in an id can't break the URL
`/records/${encodeURIComponent(id)}`
```

---

## React

### 18. Custom hooks

A function whose name starts with `use` and that calls other hooks. It lets you reuse stateful logic between components. Each call gets its own state.

| Hook | What it does |
|---|---|
| `useRecordsFilters` | Status, stage, and page in the URL; debounced search |
| `useCandidateTable` | Fetches one table page; loading/error/retry; out-of-range page |
| `useCandidateFeed` | Mobile infinite scroll; keeps the scroll position |
| `useCandidateDrawerHref` | Builds a drawer link that keeps the list query |

```ts
// hooks/useCandidateDrawerHref.ts: the smallest example
export function useCandidateDrawerHref() {
  const listQuery = toListQuery(useSearchParams());
  return (id: string) => drawerHref(listQuery, id);
}
```

```tsx
// components/candidate-list/candidate-list.tsx: components stay simple
const table = useCandidateTable({ ...filters, page, onPageOutOfRange: setPage });
const { loadMoreAreaRef, ...feed } = useCandidateFeed(filters);
```

**Interview Q:** *Custom hook vs utility function?* Use a hook only when the logic needs hooks: state, effects, or context. Pure logic goes in a normal function (see `lib/utils/candidate-urls.ts`).

**Rules of Hooks:** only call hooks at the top level (not in conditions or loops), and only from components or other hooks.

### 19. `useState` and functional updates

When the new state depends on the old state, pass a function. React gives it the latest value.

```ts
setReloadKey((k) => k + 1);                               // retry trigger
setCandidates((prev) => [...prev, ...response.data]);     // append a page
```

**Interview Q:** *Why `setCount(c => c + 1)` instead of `setCount(count + 1)`?* `count` can be stale inside async code or batched updates. The function form always gets the current value.

### 20. `useEffect`, cleanup, and dependency arrays

An effect syncs the component with something outside React: the network, the DOM, timers, or event listeners. The cleanup runs before the next run and when the component unmounts.

```ts
// components/candidate-detail/candidate-drawer.tsx
useEffect(() => {
  if (!candidateId) return;

  const onKeyDown = (event: KeyboardEvent) => {
    if (event.key === "Escape") close();
  };
  document.addEventListener("keydown", onKeyDown);

  const previousOverflow = document.body.style.overflow;
  document.body.style.overflow = "hidden";        // lock page scroll

  return () => {                                  // cleanup: undo everything
    document.removeEventListener("keydown", onKeyDown);
    document.body.style.overflow = previousOverflow;
  };
}, [candidateId, close]);
```

- `[]`: run once after mount.
- `[a, b]`: run again when `a` or `b` changes.
- No array: run after every render.

### 21. Race conditions in data fetching (the `ignore` flag)

If the user changes the filters quickly, an older, slower response can arrive last and overwrite the newer one. The cleanup marks old requests as ignored.

```ts
// hooks/useCandidateTable.ts
useEffect(() => {
  let ignore = false;               // only the latest request updates the table
  async function load() {
    const response = await getRecords({ page, /* ... */ });
    if (!ignore) setResult(response);
  }
  load();
  return () => { ignore = true; };  // a newer run makes this one stale
}, [reloadKey, page, status, stage, search]);
```

`useCandidateFeed` takes this further with `listIdRef`. "Load more" isn't an effect, so it compares a list id instead.

**Interview Q:** *How do you prevent out-of-order responses?* Use an ignore flag in the cleanup, an `AbortController`, or a data library (TanStack Query, SWR) that handles it for you.

### 22. `useRef` vs `useState`

| | `useState` | `useRef` |
|---|---|---|
| Changing it re-renders | Yes | No |
| Updates | On the next render | Immediately (`.current`) |
| Use for | What the UI shows | DOM nodes, timers, flags, previous values |

```ts
// hooks/useCandidateFeed.ts
// A ref, not state: two quick observer calls must not both start a request.
const loadingMoreRef = useRef(false);

// A DOM ref: the element the IntersectionObserver watches
const loadMoreAreaRef = useRef<HTMLDivElement>(null);
<div ref={loadMoreAreaRef}>...</div>
```

**Interview Q:** *Why would a state flag fail as a "busy" lock?* State only changes on the next render, so two calls in the same tick both see `false`.

### 23. Adjusting state during render (instead of an effect)

When a prop changes and some state must reset, do it during render by comparing with the previous value. An effect would first render the old data under the new filters, then fix it.

```ts
// hooks/useCandidateFeed.ts
const filterKey = `${status ?? ""}|${stage ?? ""}|${search ?? ""}`;
const [prevFilterKey, setPrevFilterKey] = useState(filterKey);
if (filterKey !== prevFilterKey) {
  setPrevFilterKey(filterKey);
  setCandidates([]);
  setLoading(true);
  // ...
}
```

This is the React docs' "You Might Not Need an Effect" pattern. Another option is to remount the component with a new `key`.

### 24. Derived values instead of state

If a value can be calculated from props or state, don't store it in state. Storing it creates two sources of truth that can disagree.

```ts
// hooks/useCandidateTable.ts
const isOutOfRange =
  !loading && !error && result !== null &&
  result.total > 0 && result.data.length === 0 && page > 1;
const lastPage = result ? Math.max(1, Math.ceil(result.total / PAGE_SIZE)) : 1;

// hooks/useRecordsFilters.ts
const hasActiveFilters = Boolean(search || status || stage);
```

### 25. `useCallback`

Keeps the same function reference between renders until its dependencies change. That matters when the function is an effect dependency, or a prop of a memoized child.

```ts
// components/candidate-detail/candidate-drawer.tsx
const close = useCallback(() => {
  router.push(listHref(listQuery), { scroll: false });
}, [router, listQuery]);
// `close` is in the keydown effect's dependency array; without useCallback
// the listener would be removed and re-added on every render.
```

**Interview Q:** *Should you wrap every function in `useCallback`?* No. It has a cost. Use it when the reference itself matters. The React Compiler can automate this.

### 26. `useEffectEvent`

New in React 19.2. A function called from an effect that always sees the latest props and state, but isn't a dependency. It solves the "I need the latest value, but I don't want the effect to re-run" problem.

```ts
// hooks/useCandidateFeed.ts
const onLoadMoreAreaVisible = useEffectEvent(() => {
  if (loadMoreError) return;
  loadMore();                  // always the latest loadMore / state
});

useEffect(() => {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) onLoadMoreAreaVisible();
  }, { rootMargin: "200px" });
  observer.observe(area);
  return () => observer.disconnect();
}, [loading, error, candidates.length]);   // onLoadMoreAreaVisible not listed
```

**Interview Q:** *How did people solve this before `useEffectEvent`?* By storing the latest callback in a ref (`callbackRef.current = fn`), or by adding it to the dependencies and accepting the extra re-runs.

### 27. Debouncing

Wait until the user stops typing before sending a request. Each keystroke cancels the previous timer.

```ts
// hooks/useRecordsFilters.ts
const [search, setSearch] = useState("");           // input value, every keystroke
const [searchQuery, setSearchQuery] = useState(""); // what the API gets

useEffect(() => {
  const timer = setTimeout(() => setSearchQuery(search.trim()), SEARCH_DELAY_MS);
  return () => clearTimeout(timer);   // the next keystroke cancels this one
}, [search]);
```

**Interview Q:** *Debounce vs throttle?* Debounce runs once, after the activity stops. Throttle runs at most once per interval while the activity continues.

### 28. Conditional rendering

```tsx
// components/candidate-list/candidate-list.tsx: one state at a time
{feed.loading ? (
  <CandidateListSkeleton />
) : feed.error ? (
  <ErrorState message={feed.error} onRetry={feed.retry} />
) : feed.total === 0 ? (
  emptyState
) : (
  <CandidateCards candidates={feed.candidates} />
)}
```

```tsx
// components/candidate-detail/candidate-drawer.tsx: render nothing
if (!candidateId) return null;
```

### 29. Lists and `key`

`key` tells React which item is which between renders. Use a stable id, not the array index, when items can be added, removed, or reordered.

```tsx
{candidates.map((candidate) => (
  <li key={candidate.id}>...</li>
))}
```

**Interview Q:** *What goes wrong with index keys?* After an insert or delete, React reuses the wrong element's state and DOM. Input values end up in the wrong rows.

### 30. Reusable components and default props

```tsx
// components/shared/error-state.tsx
type ErrorStateProps = {
  title?: string;
  message: string;
  onRetry?: () => void;      // optional: no button if not passed
};

export default function ErrorState({
  title = "Something went wrong",   // default value via destructuring
  message,
  onRetry,
}: ErrorStateProps) { /* ... */ }
```

The same component is used for the list, and later for details and notes.

`STATUS_STYLES: Record<CandidateStatus, string>` in `candidate-badges.tsx` maps data to styles. Lookup tables like this beat long `if` or `switch` chains.

### 31. Loading, error, and empty states

Every request has at least three outcomes besides success. Each one needs UI:

- **Loading:** a skeleton shaped like the content (`CandidateListSkeleton`).
- **Error:** a message plus Retry (`ErrorState`, with `retry` bumping `reloadKey`).
- **Empty:** two different messages. "No candidates yet" means the database is empty. "No matching candidates" means nothing matches the filters.

### 32. Strict Mode runs effects twice in development

In development, React mounts, unmounts, and mounts again, to expose missing cleanups.

```ts
// hooks/useCandidateFeed.ts
// Not cleared here: in dev React runs this effect twice, and the first
// run's cleanup cancels the frame. Clearing late lets the second run scroll.
```

**Interview Q:** *Why does my effect run twice?* Strict Mode, in development only. The fix is a correct cleanup, not removing Strict Mode.

### 33. IntersectionObserver (infinite scroll)

A browser API that reports when an element enters the viewport. It's cheaper than listening to `scroll` events.

```ts
// hooks/useCandidateFeed.ts
const observer = new IntersectionObserver(
  (entries) => { if (entries[0].isIntersecting) onLoadMoreAreaVisible(); },
  { rootMargin: "200px" },   // trigger 200px early, so cards are ready in time
);
observer.observe(area);
return () => observer.disconnect();
```

### 34. Accessibility in components

```tsx
// components/candidate-detail/candidate-drawer.tsx
<aside role="dialog" aria-modal="true" aria-labelledby="candidate-drawer-title">
  <h2 id="candidate-drawer-title">{candidate.full_name}</h2>
```

```tsx
// Icon-only buttons need a text label
<button aria-label="Close">...</button>
// Decorative SVGs are hidden from screen readers
<svg aria-hidden="true" ... />
// Errors are announced
<div role="alert">...</div>
// Table headers
<th scope="col">Name</th>
```

---

## Next.js (App Router)

### 35. File-based routing and dynamic segments

The folder structure under `app/` is the URL structure. `[id]` is a dynamic segment.

```text
app/page.tsx                  →  /
app/candidates/[id]/page.tsx  →  /candidates/12
app/layout.tsx                →  wraps every page
```

### 36. Server Components vs Client Components

- **Server Components** are the default in the App Router. They render on the server and send no JavaScript for themselves. They can't use state, effects, or browser APIs.
- **Client Components** start with `"use client"`. They can use hooks, event handlers, and `window`.

```tsx
// app/page.tsx: Server Component (no directive); composes client parts
export default function Home() { /* ... */ }

// components/candidate-list/candidate-list.tsx
"use client";   // uses state, effects, useSearchParams
```

`candidate-full-page.tsx` and `error-state.tsx` have no directive. They are rendered by whichever kind of component imports them.

**Interview Q:** *Where does `"use client"` go?* As high as necessary but as low as possible. Everything a client file imports becomes client code.

### 37. `useSearchParams`, `useRouter`, `usePathname`

These come from `next/navigation`, not `next/router`, which is the older Pages Router.

```ts
// hooks/useRecordsFilters.ts
const searchParams = useSearchParams(); // read-only; copy it to change it
const router = useRouter();
const pathname = usePathname();

const params = new URLSearchParams(searchParams.toString()); // editable copy
params.set("status", value);
router.replace(`${pathname}?${params}`, { scroll: false });
```

### 38. `router.push` vs `router.replace`

- `push` adds a history entry. Back returns to it. The project uses it for opening and closing the drawer.
- `replace` swaps the current entry. The project uses it for filter changes, so Back doesn't step through every dropdown change.
- `{ scroll: false }` keeps the scroll position.

```ts
router.push(listHref(listQuery), { scroll: false });   // drawer close
router.replace(query ? `${pathname}?${query}` : pathname, { scroll: false }); // filters
```

### 39. The URL as state

Status, stage, page, and the open drawer (`?candidate=12`) all live in the URL. That gives you refresh, shareable links, and a working Back button for free. Search text stays in local state, so the URL doesn't change on every keystroke.

```text
/?status=received&page=2                  list
/?status=received&page=2&candidate=12     list + drawer
/candidates/12?status=received&page=2     full page; the list query is kept for Back
```

Invalid values (`?status=bogus`, `?page=abc`) are validated, ignored, and then removed from the URL (`useRecordsFilters`).

### 40. `Suspense` boundaries

`<Suspense fallback>` shows a fallback while its children wait for something. In Next.js, a component that calls `useSearchParams` must sit inside a `Suspense` boundary. Otherwise the whole page falls back to client rendering.

```tsx
// app/page.tsx
<Suspense fallback={<CandidateListSkeleton />}>
  <CandidateList />
</Suspense>
```

### 41. `params` and `searchParams` are Promises

Since Next.js 15, a page's `params` and `searchParams` are Promises. Reading them inside `<Suspense>` keeps the rest of the page static, which allows prerendering.

```tsx
// app/candidates/[id]/page.tsx: not async; the promise resolves inside Suspense
<Suspense fallback={<p>Loading…</p>}>
  {Promise.all([props.params, props.searchParams]).then(
    ([{ id }, searchParams]) => (
      <CandidateFullPage id={id} listQuery={toListQuery(searchParams)} />
    ),
  )}
</Suspense>
```

`Promise.all` waits for both promises in parallel.

### 42. `PageProps` and `LayoutProps` typed helpers

These are global types generated by Next.js for each route. They type `params` from the folder name, so you don't write the type by hand.

```tsx
export default function CandidateDetailPage(props: PageProps<"/candidates/[id]">) {}
export default function RootLayout({ children }: LayoutProps<"/">) {}
```

### 43. `Link` and client-side navigation

`<Link>` navigates without a full page reload and prefetches routes. Use it for navigation, and `router.push` only for programmatic navigation, such as after an action or on Escape.

```tsx
// components/candidate-list/candidate-table.tsx
<Link
  href={getCandidateDrawerHref(candidate.id)}
  scroll={false}
  className="after:absolute after:inset-0"   // makes the whole row clickable
>
  {candidate.full_name}
</Link>
```

### 44. `cacheComponents` and `<Activity>`

With `cacheComponents: true` (in `next.config.ts`), Next.js keeps a page you navigated away from alive but hidden, inside React's `<Activity>`. State survives, but effects run again when the page shows. `useCandidateFeed` uses `loadedKeyRef` to skip refetching page 1, so the already-loaded cards and the scroll position are kept.

```ts
// hooks/useCandidateFeed.ts
const loadKey = `${filterKey}|${reloadKey}`;
if (loadedKeyRef.current === loadKey) return;   // shown again: keep the cards
```

### 45. Metadata API and `next/font`

```tsx
// app/layout.tsx
export const metadata: Metadata = {
  title: "Talent Pipeline Tracker | HealthCore",
  description: "...",
};

const geistSans = Geist({ variable: "--font-geist-sans", subsets: ["latin"] });
// self-hosted at build time: no layout shift, no request to Google at runtime
```

### 46. Environment variables (`NEXT_PUBLIC_`)

Only variables prefixed with `NEXT_PUBLIC_` reach the browser. They're inlined at build time. Never put secrets in them.

```ts
// lib/api/api-fetch.ts
const base = process.env.NEXT_PUBLIC_API_URL;
if (!base) {
  throw new Error("API URL not configured. Set NEXT_PUBLIC_API_URL in .env.local.");
}
```

`.env.local` is git-ignored. `.env.example` documents the variable.

---

## Promises and API calls

### 47. Promises and `Promise<T>`

A Promise is an object for a value that isn't ready yet, such as a network response. It is always in one of three states:

| State | Meaning | How you handle it |
|---|---|---|
| **pending** | Still waiting | Show loading |
| **fulfilled** | Finished with a value | `await` / `.then(value => …)` |
| **rejected** | Failed with an error | `try`/`catch` / `.catch(err => …)` |

Once a Promise is fulfilled or rejected, it is **settled** and never changes again.

`Promise<T>` is the TypeScript type: "a Promise that will fulfill with a `T`."

```ts
// lib/api/records.ts
export async function getRecord(id: string): Promise<Candidate> {
  const data = await apiFetch<unknown>(`/records/${encodeURIComponent(id)}`);
  if (!isCandidate(data)) {
    throw new Error("The candidate details came back in an unexpected format.");
  }
  return data; // fulfills the Promise with a Candidate
}
```

```ts
// The caller gets the Candidate itself after await, not the Promise
const candidate = await getRecord("12");   // candidate: Candidate
candidate.full_name;
```

Combining several Promises:

| Helper | Fulfills when | Rejects when |
|---|---|---|
| `Promise.all([a, b])` | all fulfill, giving an array of values | **any one** rejects (fail fast) |
| `Promise.allSettled([a, b])` | all settle, giving a status for each | never |
| `Promise.race([a, b])` | the first one settles, with its result | the first one settles, with its error |
| `Promise.any([a, b])` | the first one fulfills | all reject |

```tsx
// app/candidates/[id]/page.tsx: wait for params and searchParams in parallel
Promise.all([props.params, props.searchParams]).then(([{ id }, searchParams]) => /* ... */);
```

Step 6 will use `Promise.all` to load a record and its notes at the same time.

**Interview Q:** *Why `Promise.all` instead of two `await`s in a row?* Two `await`s run one after the other, so the total time is A + B. `Promise.all` runs them in parallel, so the total time is roughly the longer of the two. Use sequential `await`s only when B needs A's result.

**Interview Q:** *`Promise.all` vs `Promise.allSettled`?* `all` fails as soon as one Promise fails. `allSettled` waits for every one and reports each result. Use it when partial success is fine, for example showing the record even if the notes failed to load.

### 48. An `async` function always returns a Promise

- Whatever you `return` inside an `async` function is wrapped in a Promise automatically.
- Whatever you `throw` turns into a **rejected** Promise.
- That's why the return type of `getRecord` is `Promise<Candidate>`, even though the code returns a plain `Candidate`.

```ts
async function getRecord(id: string): Promise<Candidate> {
  // ...
  throw new Error("...");   // the caller's `await` throws → caught by try/catch
  return data;              // the caller's `await` gets a Candidate
}
```

`await` pauses only the function it's in, not the whole page. Other code and rendering keep running.

**Interview Q:** *What does `async function f() { return 1 }` return?* `Promise<number>`, not `1`.

**Interview Q:** *What happens if you forget `await`?* You get the Promise itself, not the value. `data.full_name` is then `undefined`, and the `isCandidate` check would fail, because a Promise isn't a candidate. Errors also escape your `try`/`catch`, since nothing waits for the Promise to reject.

### 49. Generic syntax: `<T>`, not `(T)`

Type parameters always go in **angle brackets**. Parentheses are for values: function calls and parameters.

```ts
Promise<Candidate>             // ✅ a type with a type parameter
Promise(Candidate)             // ❌ looks like calling a function → syntax error

apiFetch<unknown>("/records")  // <type arguments> then (value arguments)
useState<string | null>(null)
```

A real mistake from this project: `getRecord(id: string): Promise(Candiate)` gave `error TS1144: '{' or ';' expected` and `TS1005: '=>' expected`. TypeScript read `Promise(` as a function call, so the error message didn't mention generics at all. A misspelled type (`Candiate`) also fails, with "Cannot find name".

**Interview Q:** *Read this type: `Promise<Record<string, Candidate[]>>`.* A Promise that fulfills with an object whose keys are strings and whose values are arrays of candidates. Read it from the outside in.

### 50. HTTP methods: GET, POST, PUT, PATCH, DELETE

The REST endpoints in this project:

| Method | Endpoint | Purpose | Body | Safe | Idempotent |
|---|---|---|---|---|---|
| GET | `/records`, `/records/:id` | Read | No | Yes | Yes |
| POST | `/records`, `/records/:id/notes` | Create | Yes | No | **No** |
| PUT | `/records/:id` | **Replace** the whole resource | Full object | No | Yes |
| PATCH | `/records/:id` | **Change some** fields | Only the changed fields | No | Usually |
| DELETE | `/records/:id/notes/:note_id` | Remove | Usually none | No | Yes |

- **Safe:** it doesn't change anything on the server.
- **Idempotent:** sending the same request twice has the same effect as sending it once. Two identical POSTs create two candidates. Two identical PUTs leave the same result.

```ts
// GET: the default method, so no options are needed
apiFetch<unknown>(`/records/${encodeURIComponent(id)}`);

// PATCH: only the fields that changed
apiFetch<unknown>(`/records/${encodeURIComponent(id)}`, {
  method: "PATCH",
  body: JSON.stringify(payload),        // e.g. { "status": "selected" }
});
```

```ts
// types/candidate.ts: the payload types match the method
type UpdateCandidateStatusPayload = Partial<Pick<Candidate, "status" | "stage">>; // PATCH
type UpdateCandidatePayload = Omit<Candidate, "id" | "status" | /* ... */>;      // PUT
```

**Interview Q:** *PUT vs PATCH?* PUT sends the complete replacement, and any field you leave out may be cleared. PATCH sends only what changes. This project uses PATCH for the status/stage dropdowns and PUT for the full Edit form.

**Interview Q:** *Why does idempotency matter?* Requests can be retried safely after a timeout only if they're idempotent. A retried POST can create duplicates, which is one reason forms disable the Save button while saving.

**Interview Q:** *Why `JSON.stringify(payload)`?* `fetch` sends the body as text. The object must be serialized, and the `Content-Type: application/json` header tells the server how to read it. `apiFetch` adds that header automatically when there's a body.

### 51. A typed API layer (one function per endpoint)

Components never call `fetch` directly. Each endpoint gets one small function. They all go through one shared helper and return a checked, typed result.

```text
component  →  getRecord(id)        lib/api/records.ts          one function per endpoint
           →  apiFetch(path, opts) lib/api/api-fetch.ts        URL, headers, HTTP + JSON errors
           →  isCandidate(data)    lib/api/validators/         runtime shape check
```

```ts
// lib/api/records.ts: every endpoint function has the same shape
export async function getRecord(id: string): Promise<Candidate> {
  const data = await apiFetch<unknown>(`/records/${encodeURIComponent(id)}`); // 1. request
  if (!isCandidate(data)) throw new Error("...");                             // 2. validate
  return data;                                                                // 3. typed result
}
```

Benefits:
- **One place to change:** a new base URL, auth header, or error format changes `apiFetch` only.
- **Components stay simple:** they work with `Candidate`, not URLs or JSON.
- **Testable:** you can mock `getRecord` without touching the network.
- **Consistent errors:** network failures, HTTP errors, invalid JSON and wrong shapes all become an `Error` with a readable message, which `ErrorState` displays.

**Interview Q:** *Why not call `fetch` inside components?* It duplicates URL building, headers and error handling. It mixes data access with UI, and it leaves response types unchecked. Libraries like TanStack Query sit on top of a layer like this, adding caching, retries and request deduplication.

---

## Timers and save feedback

### 52. Timers in effects: auto-dismiss vs debounce vs throttle

All three use `setTimeout`, and the code can look almost the same. The difference is **what starts the timer and what it's for.**

**Auto-dismiss** is a one-shot timer. Something happens once, and the timer undoes it later. Here, a success message hides itself after 3 seconds:

```ts
// lib/constants.ts
export const SUCCESS_MESSAGE_MS = 3000;

// components/candidate-detail/recruitment-progress-card.tsx
// (the same code is in general-information-card.tsx)
const [saved, setSaved] = useState(false);

// Hide "Saved" after a few seconds
useEffect(() => {
  if (!saved) return;                                  // nothing showing: no timer
  const timer = setTimeout(() => setSaved(false), SUCCESS_MESSAGE_MS);
  return () => clearTimeout(timer);                    // cleanup (see #53)
}, [saved]);

// after a successful PATCH:
setSaved(true);   // shows the banner → the effect starts the 3 s timer
```

```tsx
{saved && <SuccessMessage message="Recruitment progress saved." />}
```

```tsx
// general-information-card.tsx: after a successful PUT the form closes,
// so the message shows in view mode, under the updated details
<>
  <CandidateDetails candidate={candidate} layout={layout} />
  {saved && <SuccessMessage message="General information saved." />}
</>
```

**Debounce** waits until a stream of events stops, then runs once. Each new event cancels the previous timer. Here, a search request is sent only after typing pauses (see #27):

```ts
// hooks/useRecordsFilters.ts
useEffect(() => {
  const timer = setTimeout(() => setSearchQuery(search.trim()), SEARCH_DELAY_MS);
  return () => clearTimeout(timer);   // the next keystroke cancels this timer
}, [search]);
```

| | Auto-dismiss | Debounce | Throttle |
|---|---|---|---|
| Triggered by | One event (a save succeeded) | Many rapid events (keystrokes) | Many rapid events (scroll, resize) |
| Runs | Once, a fixed time after the event | Once, after the events **stop** | At most once per interval, **while** the events continue |
| `clearTimeout` is for | Cleanup if the component unmounts or `saved` changes | Cancelling the previous event's timer (the core of debounce) | Not needed; a "last run" time is checked instead |
| Example here | "Recruitment progress saved." disappears after 3 s | Search waits 300 ms after the last keystroke | (not used) |

The cleanup line, `return () => clearTimeout(timer)`, is the same in both effects. In the debounce it **is** the logic: every keystroke re-runs the effect, so the previous timer is cancelled. In auto-dismiss it's only a safety net.

**Interview Q:** *Is this auto-hide effect a debounce?* No. A debounce collapses many events into one call after they stop. This effect reacts to one state change and schedules one follow-up. The shape is similar (`setTimeout` + `clearTimeout` in an effect), but the purpose is different.

**Interview Q:** *Debounce vs throttle for a search box?* Debounce: you want one request after the user stops typing. Throttle fits continuous events where you want regular updates, such as scroll position or resize.

### 53. Effect cleanup for timers

If a timer can outlive the component, always clear it in the effect's cleanup.

```ts
useEffect(() => {
  if (!saved) return;
  const timer = setTimeout(() => setSaved(false), SUCCESS_MESSAGE_MS);
  return () => clearTimeout(timer);
}, [saved]);
```

The cleanup runs in two situations:
1. **Before the effect runs again,** when `saved` changes. If `saved` goes back to `false` first (for example, the user clicks Edit), the old timer is cancelled, so it can't hide a later message too early.
2. **When the component unmounts,** for example when the drawer closes or another candidate opens. The timer is cancelled, so nothing tries to update a component that's gone.

The `if (!saved) return;` line means no timer is started when there's nothing to hide.

Both cards also reset `saved` when Edit is opened, which triggers the cleanup above:

```ts
const startEditing = () => {
  setError(null);
  setSaved(false); // no old "saved" message next to the open form
  setEditing(true);
};
```

**Interview Q:** *What happens if you don't clear a timer?* It still fires later. It may update state the user has already moved past (hiding a newer message too early), or try to update an unmounted component. With `setInterval`, the leak repeats forever. Modern React no longer warns about updates after unmount, but the bug in your logic remains.

**Interview Q:** *Why not call `setTimeout` straight after `setSaved(true)` in the save function?* It would work, but nothing would cancel it on unmount, and you'd have to manage the timer ID yourself (for example in a ref). Putting it in an effect that depends on `saved` ties the timer's lifetime to the state it controls.

**Interview Q:** *The same `saved` state + timer effect is now in two components. What would you do?* Extract a custom hook (see #18), e.g. `useAutoDismiss(ms)` returning `[visible, show, hide]`. Each component keeps its own state (hooks share logic, not state), and the timer and cleanup are written once. A common rule of thumb is to extract on the second or third copy, not before.

### 54. Save feedback: saving, success, error

Every save needs three kinds of feedback, plus protection against sending it twice.

```ts
// components/candidate-detail/recruitment-progress-card.tsx
const save = async () => {
  const payload: UpdateCandidateStatusPayload = {};
  if (status !== candidate.status) payload.status = status;  // only what changed
  if (stage !== candidate.stage) payload.stage = stage;
  if (Object.keys(payload).length === 0) {
    setEditing(false);                                       // nothing changed: no request
    return;
  }

  setSaving(true);                    // 1. saving: disables the form
  setError(null);
  try {
    const updated = await updateRecordStatus({ id: candidate.id, payload });
    onSaved(updated);                 // show the server's version everywhere
    setEditing(false);
    setSaved(true);                   // 2. success: green banner, auto-hides
  } catch (err) {
    setError(err instanceof Error ? err.message : "Could not save the changes.");
                                      // 3. error: stay in edit mode, keep the user's choices
  } finally {
    setSaving(false);
  }
};
```

```tsx
<fieldset disabled={saving}>...</fieldset>          {/* disables every control inside */}
<button type="submit" disabled={saving}>
  {saving ? "Saving…" : "Save"}
</button>
{error && <ErrorState message={error} onRetry={save} />}
```

| State | What the user sees | Why |
|---|---|---|
| Saving | Disabled controls, "Saving…" | No double submits; shows something is happening |
| Success | Green banner, which hides after 3 s; badges update | Confirms the change; no reload needed |
| Error | `ErrorState` with Retry; the user's choices are kept | The user can retry without re-entering anything |

Using the **server's response** (`updated`) instead of the local draft means the UI shows exactly what was saved, including server-set fields like `updated_at`.

**Interview Q:** *Optimistic vs pessimistic updates?* **Pessimistic** (used here): wait for the server, then update the UI. It's simple and always accurate, but it feels slower. **Optimistic**: update the UI immediately, then roll back if the request fails. It feels instant, but needs rollback logic. Use optimistic updates for frequent, low-risk actions (likes); use pessimistic updates for important decisions (hiring status).

**Interview Q:** *How do you prevent a form from being submitted twice?* Disable the submit button and inputs while saving (`<fieldset disabled>`). For extra safety, check a "busy" ref at the start of the handler (see #22), since a ref updates immediately.

---

## Forms and validation

### 55. `keyof`, `Record`, and `Partial` together: a typed errors object

Three type tools, combined to describe "at most one error message per form field":

```ts
// types/candidate.ts
export type CandidateFormValues = {
  full_name: string;
  email: string;
  phone: string;
  position: string;
  linkedin_url: string;
  cv_url: string;
  experience_years: string;
};

// One message per invalid field; a field with no entry is valid
export type CandidateFormErrors = Partial<
  Record<keyof CandidateFormValues, string>
>;
```

Read it from the inside out:

| Step | Type | Result |
|---|---|---|
| 1. `keyof` | `keyof CandidateFormValues` | The union of its key names: `"full_name" \| "email" \| "phone" \| ...` |
| 2. `Record` | `Record<that union, string>` | An object with **every** one of those keys, each holding a `string` |
| 3. `Partial` | `Partial<that object>` | The same keys, but **all optional** (`full_name?: string`, ...) |

- **`keyof T`** turns an object type into a union of its keys. It is a type-level operator, not a utility type, and is often saved as its own alias:

  ```ts
  // components/candidate-form/candidate-form.tsx
  type FieldName = keyof CandidateFormValues;
  const inputId = (name: FieldName) => `${idPrefix}-${name}`;
  ```

  Passing `"emial"` is now a compile error.
- **`Record<K, V>`** builds an object type from a set of keys and one value type (see #2).
- **`Partial<T>`** makes every key optional (see #2). Without it, the validator would have to return a message for every field, even valid ones.

Why this shape fits validation: a missing key means "valid", so `{}` means the whole form is fine, and checking is a simple lookup.

```ts
// lib/utils/candidate-form.ts: start empty, add a key only for a bad field
const errors: CandidateFormErrors = {};
if (!values.full_name.trim()) {
  errors.full_name = "Enter the candidate's full name.";
}
return errors;
```

```tsx
// candidate-form.tsx: look a field up by name; undefined = no error
const error = errors[field.name];
{error && <p id={errorId}>{error}</p>}
```

Because the keys come from `CandidateFormValues` via `keyof`, adding a field to the form type adds it to the errors type automatically. A typo like `errors.fullname` is also a compile error.

**Interview Q:** *What does `keyof` return?* A union of string literal types, one per property name. `keyof { a: string; b: number }` is `"a" | "b"`.

**Interview Q:** *`Partial<Record<K, V>>` vs `Record<K, V>`?* `Record` requires every key; `Partial<Record>` allows any subset. Use `Record` when every key must exist (like `STATUS_LABELS`, a label for every status); use `Partial<Record>` when only some keys are present (like errors, only for invalid fields).

**Interview Q:** *How would you write `Partial` yourself?* It's a mapped type: `type MyPartial<T> = { [K in keyof T]?: T[K] }`. It loops over each key of `T` (`K in keyof T`), adds `?`, and keeps the original value type (`T[K]`, an indexed access type). `Record<K, V>` is similar: `{ [P in K]: V }`.

---

## Sharing state across components

### 56. The Context API (`createContext`, provider, `useContext`)

Context shares a value with every component inside a provider, without passing props through each level ("prop drilling"). Use it when components that need the same value are far apart in the tree, or not parent and child at all.

Here, the list and the drawer are **siblings**, and the full page is a **different route**, so no prop can carry "a candidate was saved" from one to the other:

```
app/layout.tsx
└── CandidateUpdatesProvider        ← holds saveCount
    ├── /                 → CandidateList (reads) + CandidateDrawer (saves)
    └── /candidates/[id]  → CandidateFullPage (saves)
```

Three parts: create the context, provide a value, read it.

```tsx
// context/candidate-updates.tsx
"use client"; // context and useState only work in Client Components

type CandidateUpdates = {
  saveCount: number;                // goes up by one after every save
  notifyCandidateSaved: () => void; // call after a successful save
};

// 1. Create. Default null = "no provider above this component"
const CandidateUpdatesContext = createContext<CandidateUpdates | null>(null);

// 2. Provide: owns the state, wraps the part of the tree that can read it
export function CandidateUpdatesProvider({ children }: { children: ReactNode }) {
  const [saveCount, setSaveCount] = useState(0);
  const notifyCandidateSaved = useCallback(() => {
    setSaveCount((count) => count + 1);
  }, []);
  const value = useMemo(
    () => ({ saveCount, notifyCandidateSaved }),
    [saveCount, notifyCandidateSaved],
  );
  // React 19: <Context value> (older React: <Context.Provider value>)
  return <CandidateUpdatesContext value={value}>{children}</CandidateUpdatesContext>;
}

// 3. Read: a custom hook that fails loudly if the provider is missing
export function useCandidateUpdates(): CandidateUpdates {
  const context = useContext(CandidateUpdatesContext);
  if (!context) {
    throw new Error("useCandidateUpdates must be used inside CandidateUpdatesProvider.");
  }
  return context; // narrowed from CandidateUpdates | null to CandidateUpdates
}
```

```tsx
// app/layout.tsx: a Server Component can render a Client Component provider
<body>
  <CandidateUpdatesProvider>{children}</CandidateUpdatesProvider>
</body>
```

```ts
// hooks/useCandidateDetail.ts: the writer
const { notifyCandidateSaved } = useCandidateUpdates();
// after a successful PATCH/PUT:
notifyCandidateSaved();

// components/candidate-list/candidate-list.tsx: the reader
const { saveCount } = useCandidateUpdates();
```

**Interview Q:** *When would you use context instead of props?* When many components, or distant ones, need the same value: theme, the signed-in user, locale, or a cross-page signal like this one. For a value one or two levels down, props are simpler and clearer.

**Interview Q:** *Why wrap `useContext` in a custom hook?* One import for callers, a clear error if the provider is missing (instead of a confusing `null` bug), and `null` is removed from the type, so callers don't need checks.

**Interview Q:** *Can you use context in a Next.js Server Component?* No. Context needs client-side React. Make the provider a Client Component (`"use client"`) that takes `children`, and render it from the Server Component layout. The pages passed in as `children` can still be Server Components.

**Interview Q:** *Context vs a state library (Redux, Zustand)?* Context is built in and fine for values that change rarely. Every component that reads a context re-renders when its value changes, so large, frequently changing state is better in a store with selectors, where a component only re-renders for the slice it uses.

### 57. `useMemo` and `useCallback` for a context value

- `useMemo(() => value, deps)` keeps the same **value** between renders until a dependency changes.
- `useCallback(fn, deps)` keeps the same **function** between renders (see #25). It is `useMemo(() => fn, deps)` written shorter.

This matters for a context because React compares its value with `Object.is`. A new object, even with the same contents, makes **every** component that reads the context re-render.

```tsx
// Without useMemo: a brand-new object on every provider render,
// so every reader re-renders each time, even when nothing changed
<CandidateUpdatesContext value={{ saveCount, notifyCandidateSaved }}>

// With useMemo: the same object until saveCount actually changes
const value = useMemo(
  () => ({ saveCount, notifyCandidateSaved }),
  [saveCount, notifyCandidateSaved],
);
```

`useMemo` only helps if its dependencies are stable too. That's why `notifyCandidateSaved` uses `useCallback` with `[]`. The functional update (`count => count + 1`) means it never reads `saveCount`, so it never has to change:

```tsx
const notifyCandidateSaved = useCallback(() => {
  setSaveCount((count) => count + 1); // always adds to the latest value
}, []);
```

| | `useMemo` | `useCallback` |
|---|---|---|
| Returns | The result of calling the function | The function itself |
| Use for | An object/array passed to a context, a prop, or a dependency array; an expensive calculation | A function passed to a context, a prop, or a dependency array |
| Here | The context `value` object | `notifyCandidateSaved` |

**Interview Q:** *Difference between `useMemo` and `useCallback`?* `useMemo` caches a value; `useCallback` caches a function. `useCallback(fn, deps)` is the same as `useMemo(() => fn, deps)`.

**Interview Q:** *Why memoize a context value?* Readers re-render whenever the value is a new reference. An inline `{...}` object is new on every render, so every reader would re-render even when nothing changed.

**Interview Q:** *Should you always use `useMemo`?* No. It has a cost and makes code harder to read. Use it when the **reference** matters (context values, props of memoized children, effect dependencies) or the calculation is genuinely slow. The React Compiler can add this memoization automatically.

### 58. Signal vs data: refetch after a save instead of patching

After a save, there are two ways to update another list:

| | Patch local state | Refetch (used here) |
|---|---|---|
| How | Swap the saved candidate into the list's array | Tell the list "something changed"; it asks the API again |
| Requests | None | One per loaded page |
| Filters | A candidate that no longer matches stays until the next reload | The API applies the filters, so it drops out |
| Source of truth | Each list's copy | The server |

This project sends a **signal**, not data. The context only holds a counter; each list fetches fresh data itself:

```ts
// hooks/useCandidateTable.ts (desktop): saveCount is an effect dependency
// but not part of requestKey, so the old rows stay until new ones arrive (no skeleton)
}, [reloadKey, page, status, stage, search, saveCount]);
```

```ts
// hooks/useCandidateFeed.ts (mobile): reload every page already shown,
// in parallel, so the extra pages and the scroll position are kept
const pages = await Promise.all(
  Array.from({ length: lastPage }, (_, index) =>
    getRecords({ page: index + 1, limit: PAGE_SIZE, status, stage, search }),
  ),
);
setCandidates(pages.flatMap((response) => response.data));
```

**Interview Q:** *After a mutation, refetch or update the cache manually?* Refetching is simpler and always correct, especially with server-side filters, sorting, or pagination; it costs extra requests. Manual cache updates are instant and cheaper, but you must re-apply every rule the server would. Libraries like TanStack Query offer both: `invalidateQueries` (refetch) and `setQueryData` (manual update).

**Interview Q:** *Why put a counter in context instead of the saved candidate?* The list doesn't need the candidate; it needs to know its data is out of date. A counter that changes on every save is the simplest value to put in an effect's dependency array.

### 59. A type-safe generic setter: `K extends keyof T`, `T[K]`, `NonNullable`

One `setDraft` function updates either part of a candidate's draft, and TypeScript checks that the values match the part you name. Several type features work together:

```ts
// context/candidate-drafts.tsx
export type CandidateDraft = {
  general?: CandidateFormValues;                   // General information form
  progress?: Pick<Candidate, "status" | "stage">;  // Recruitment progress form
};
type DraftPart = keyof CandidateDraft; // "general" | "progress"

type CandidateDrafts = {
  drafts: Record<string, CandidateDraft>;
  setDraft: <K extends DraftPart>(
    id: string,
    part: K,
    values: NonNullable<CandidateDraft[K]>,
  ) => void;
  clearDraft: (id: string, part: DraftPart) => void;
};
```

| Feature | In the code | What it does |
|---|---|---|
| Optional properties (`?`) | `general?:`, `progress?:` | A candidate can have one draft, both, or none. The type of `general` becomes `CandidateFormValues \| undefined`. |
| `Pick<T, K>` | `Pick<Candidate, "status" \| "stage">` | Reuses the two fields from `Candidate` (see #2), so the draft can't drift from the real type |
| `keyof` | `keyof CandidateDraft` | The union of the keys: `"general" \| "progress"` (see #55) |
| `Record<K, V>` | `Record<string, CandidateDraft>` | An object used as a map: candidate ID → draft |
| Generic function type | `<K extends DraftPart>(...) => void` | `setDraft` gets its own type parameter `K` each time it's called |
| Generic constraint (`extends`) | `K extends DraftPart` | `K` can only be `"general"` or `"progress"`, not any string |
| Indexed access type | `CandidateDraft[K]` | "The type of property `K`": `CandidateDraft["progress"]` is `{ status, stage } \| undefined` |
| `NonNullable<T>` | `NonNullable<CandidateDraft[K]>` | Removes `null` and `undefined`, so you must pass real values, not `undefined` |

**How `K` links the arguments:** TypeScript infers `K` from the `part` you pass, then uses the same `K` to decide what `values` must be.

```ts
// recruitment-progress-card.tsx
setDraft(candidate.id, "progress", { status: "selected", stage: "review" }); // ✅ K = "progress"

// general-information-card.tsx
setDraft(candidate.id, "general", toCandidateFormValues(candidate));          // ✅ K = "general"

setDraft(candidate.id, "progress", toCandidateFormValues(candidate)); // ❌ form values aren't { status, stage }
setDraft(candidate.id, "notes", {});                                  // ❌ "notes" isn't a DraftPart
setDraft(candidate.id, "progress", undefined);                        // ❌ blocked by NonNullable
```

Without the generic, the best you could write is `part: DraftPart, values: CandidateDraft[DraftPart]`. That type is "either kind of values", so `setDraft(id, "progress", formValues)` would compile even though it's wrong. The generic ties `values` to the **specific** `part` given.

Reading a draft uses optional chaining (see #10), because both the candidate's entry and the part may be missing:

```ts
const draft = drafts[candidate.id]?.progress; // { status, stage } | undefined
const editing = draft !== undefined;
```

**Interview Q:** *What does `K extends keyof T` mean in a generic?* `K` is a type parameter limited to the keys of `T`. Inside the function you can then use `T[K]` safely, and callers can only pass real keys. It's the standard pattern for typed getters and setters, like `function get<T, K extends keyof T>(obj: T, key: K): T[K]`.

**Interview Q:** *What is an indexed access type?* `T[K]` looks up the type of property `K` in `T`, just like `obj[key]` looks up a value. With a union of keys it gives a union of types: `Candidate["status" | "stage"]` is `CandidateStatus | CandidateStage`.

**Interview Q:** *What does `NonNullable<T>` do?* It removes `null` and `undefined` from `T`. Here the draft parts are optional (`T | undefined`), but when you set one you must pass an actual value. Built in, it's `T & {}`.

**Interview Q:** *Why not `part: string` and `values: any`?* It would compile, but typos and wrong shapes would only fail at runtime. With `K extends DraftPart` and `CandidateDraft[K]`, both mistakes are compile errors, and the editor autocompletes `"general"` and `"progress"`.

---

<!-- Keep adding new concepts below as the project grows (notes API, forms, PATCH/PUT, focus trap, etc.). -->
