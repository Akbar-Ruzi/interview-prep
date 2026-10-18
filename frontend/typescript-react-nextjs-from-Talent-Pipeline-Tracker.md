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

// hooks/useCandidatePage.ts
type CandidatePageOptions = ListFilters & {
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
// hooks/useCandidatePage.ts
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
| `useCandidatePage` | Fetches one table page; loading/error/retry; out-of-range page |
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
const table = useCandidatePage({ ...filters, page, onPageOutOfRange: setPage });
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
// hooks/useCandidatePage.ts
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
// hooks/useCandidatePage.ts
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

<!-- Keep adding new concepts below as the project grows (notes API, forms, PATCH/PUT, focus trap, etc.). -->
