# Interview Notes: Next.js Rendering and Config

> Every snippet is from my project (StackMeet, a developer events app: Next.js 16, React 19, TypeScript, Mongoose 8, Auth.js v5), shortened where needed.
> Snippets marked **Planned fix** show the fix I would make. They aren't in the project yet.
> For each concept: what it is, the project code, and a likely interview question.
> Other notes from this project: [Next.js API and server code](../backend/nextjs-api-from-StackMeet.md), [Authentication and authorization](../auth/auth-from-StackMeet.md) and [MongoDB and Mongoose](../databases/mongodb-mongoose-from-StackMeet.md).

## Contents

- [TypeScript](#typescript)
  1. The `satisfies` operator
  2. A generic filter function: `<T extends Shape>(items: T[]): T[]`
  3. Search filter logic: normalize input, OR inside a filter, AND between filters
  4. Upcoming events: compare date strings, sort with `localeCompare`
- [Filters in the URL](#filters-in-the-url)
  5. Filter pills as links: `URLSearchParams`, `?mode=` and `aria-current`
- [Next.js rendering and config](#nextjs-rendering-and-config)
  6. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"
  7. Using `rewrites` as a reverse proxy for analytics
- [React hooks](#react-hooks)
  8. `useDropdownMenu`: click outside, focus and arrow keys in a custom hook
- [React components](#react-components)
  9. `AuthPanel`: one form for two tabs, a typed input wrapper, and an auth check first

---

## TypeScript

### 1. The `satisfies` operator

`satisfies` checks a value against a type without changing the value's inferred type. A wrong or misspelled field is a compile error.

```ts
return { success: false, message: 'Email is required.' } satisfies CreateBookingResult;
```

| | Checks the shape | Keeps the specific inferred type |
|---|---|---|
| `const x: T = value` | Yes | No, `x` becomes `T` |
| `value as T` | Only loosely, and can hide mistakes | No, treated as `T` |
| `value satisfies T` | Yes | Yes |

**Interview Q:** *Why use `satisfies` instead of `as`?* `as` tells the compiler to trust you and can hide mistakes. `satisfies` makes the compiler verify the value.

### 2. A generic filter function: `<T extends Shape>(items: T[]): T[]`

The home page search filters events. The function only needs a few fields, but it has to return the **same type** it was given, whether that's a full event from the database or a smaller card-sized object.

```ts
// lib/event-search.ts
import type { EventFields } from "@/database"; // type only: erased from the compiled JS

export type EventSearch = { q?: string; where?: string };

type SearchableEvent = Pick<EventFields, "title" | "location" | "mode" | "tags">;

export function filterEvents<T extends SearchableEvent>(events: T[], { q, where }: EventSearch): T[] {
  // ...returns events.filter(...)
}
```

Reading the signature piece by piece:

| Part | What it means |
|---|---|
| `<T ...>` | A type parameter. `T` is filled in at each call from the argument's type. |
| `extends SearchableEvent` | A constraint: `T` must have at least `title`, `location`, `mode` and `tags`. Extra fields are allowed. |
| `Pick<EventFields, ...>` | A utility type that builds a type from just those fields of `EventFields`. It only changes what TypeScript checks: real objects keep all their fields at runtime. |
| `import type` | Imports only a type. It's removed from the compiled JavaScript, so it can't pull server code (like the Mongoose model file) into the bundle. |
| `events: T[]` | An array of whatever type was passed in. |
| `{ q, where }: EventSearch` | Destructuring in the parameter list: the second argument is an object, and `q` and `where` come straight out of it. |
| `: T[]` | The return type. The caller gets back the same type it passed in. |

**Why not just `(events: SearchableEvent[]): SearchableEvent[]`?** That compiles, but the result would only know the four picked fields. The caller would lose `slug` and `date`, and `<EventCard {...event} />` would fail to type-check. With the generic, `filterEvents(cards, search)` returns the cards' own type, all fields included.

```ts
const cards: CardEvent[] = /* title, slug, date, time, mode, location, tags */;
const result = filterEvents(cards, { q: "react" }); // CardEvent[], slug and date still there
filterEvents([{ title: "x" }], {});                 // error: missing location, mode, tags
```

**Interview Q:** *What does `<T extends X>` do in a function signature?* It declares a type parameter limited to types that have at least `X`'s shape. Inside the function you can use `X`'s fields safely, and the return type can still be the caller's exact type `T`. That's how a reusable helper keeps full type information.

**Interview Q:** *Generic vs. using the base type directly?* With the base type, everything that comes out is widened to the base type and the extra fields are lost. With a generic, the input type flows through to the output.

### 3. Search filter logic: normalize input, OR inside a filter, AND between filters

The search form has two boxes, **what** and **where**. The body of `filterEvents` (#2) follows a pattern that fits most search and filter UIs.

**Step 1: normalize the input.** Users type `"  #React "`, but the data has `react`. Clean both sides the same way before comparing.

```ts
// lib/event-search.ts
const clean = (value?: string) => value?.trim().toLowerCase().replace(/^#/, "") ?? "";
```

| Piece | Does | Example |
|---|---|---|
| `value?.` | Optional chaining: if `value` is `undefined`, stop and return `undefined` instead of crashing | `clean(undefined)` doesn't throw |
| `.trim()` | Removes spaces at the start and end | `"  React "` → `"React"` |
| `.toLowerCase()` | Makes matching case-insensitive | `"React"` → `"react"` |
| `.replace(/^#/, "")` | Regex: `^` is "start of string", so only a **leading** `#` is removed | `"#ai"` → `"ai"`, `"c#"` stays `"c#"` |
| `?? ""` | Nullish coalescing: if the result is `undefined`, use `""` | `clean(undefined)` → `""` |

Always returning a string means the rest of the code never has to check for `undefined`. (`?.` and `??` are covered in more depth in [Talent Pipeline notes #10](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#10-optional-chaining-and-nullish-coalescing).)

**Step 2: each filter is a boolean built with `||`, and the results are combined with `&&`.**

```ts
return events.filter((event) => {
  const matchesWhat =
    !what ||                                                     // empty box: everything matches
    event.title.toLowerCase().includes(what) ||                  // title contains it, OR
    (event.tags ?? []).some((tag) => tag.toLowerCase().includes(what)); // any tag contains it

  const matchesWhere =
    !place ||
    (place === "online"
      ? event.mode !== "in-person"                               // "online": check the format only (online or hybrid)
      : (event.location ?? "").toLowerCase().includes(place));   // anything else: check the city

  return matchesWhat && matchesWhere;                            // must pass BOTH boxes
});
```

- **`!what ||` first:** `""` is falsy, so an empty box makes the whole condition `true` and that filter is skipped. Without it, `"anything".includes("")` happens to be `true` anyway, but the guard states the intent and short-circuits the work.
- **`||` inside a filter:** different ways to match the same box. Any one is enough.
- **Ternary (`? :`) inside a filter:** when the input decides *which* check applies, choose one branch instead of OR-ing them together (see the bug below).
- **`&&` between filters:** each box narrows the results further. A React event in New York, searched with `what=react` and `where=online`, is **excluded**, because `matchesWhere` is false.
- **`.filter()`** returns a new array of the items whose callback returned `true`. The original array is never changed.
- **`.some()`** returns `true` as soon as one element passes, and stops checking there. Its partner `.every()` needs all elements to pass.
- **`?? []` and `?? ""`:** `tags` and `location` are optional, so they get a safe default before calling `.some()` or `.toLowerCase()`.

**A bug I found and fixed: `||` where a ternary was needed.** My first version OR-ed all three checks:

```ts
// Before: "online" could still fall through to the city check
!place ||
(place === "online" && event.mode !== "in-person") ||
(event.location ?? "").toLowerCase().includes(place);
```

With `where=online`, an **in-person** event in a city like "Onlineville, TX" failed the format check, then passed the city check, so it showed up as "online". The fix makes the two checks exclusive: `"online"` means check the format, anything else means check the city, never both. Free-text search often has edge cases like this, so it's worth asking "can a keyword accidentally match a different field?"

**Interview Q:** *When should a filter use a ternary instead of `||`?* Use `||` when any of several checks may match the same input. Use a ternary when the input itself decides which single check applies, because `||` lets a value that fails the intended check slip through another one.

**Step 3: read the URL safely.** In Next.js, `searchParams.q` is typed `string | string[] | undefined`, because a URL can repeat a key: `?q=a&q=b`.

```ts
export const firstParam = (value: string | string[] | undefined) =>
  Array.isArray(value) ? value[0] : value; // string | undefined
```

`Array.isArray` is a type guard. In the `true` branch TypeScript knows `value` is `string[]`, and in the other branch it's `string | undefined`. That's type narrowing, also covered in [Talent Pipeline notes #4](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#4-type-narrowing).

**Interview Q:** *How would you combine several optional search filters?* Normalize each input, make each filter `!input || (ways it can match)`, then combine the filters with `&&`. An empty filter passes everything, so users can fill in any combination of boxes.

**Interview Q:** *`.some()` vs `.every()` vs `.filter()`?* `.some()` returns a boolean: does at least one element pass? It stops at the first match. `.every()` returns a boolean: do all elements pass? It stops at the first failure. `.filter()` returns a new array of every element that passes.

**Interview Q:** *Why is `searchParams.q` typed `string | string[] | undefined`?* A parameter can be missing, appear once, or repeat (`?tag=a&tag=b`). Code has to narrow it to the shape it expects before using it.

### 4. Upcoming events: compare date strings, sort with `localeCompare`

`getUpcomingEvents` does two things: it keeps only events happening today or later, then sorts them from earliest to latest. The whole function:

```ts
type DatedEvent = { date: string; time: string }; // "2026-11-07", "14:00"

export function getUpcomingEvents<T extends DatedEvent>(
  events: T[],
  today = new Date().toISOString().slice(0, 10)
): T[] {
  return events
    .filter((event) => event.date >= today)
    .sort((a, b) => `${a.date} ${a.time}`.localeCompare(`${b.date} ${b.time}`));
}
```

**Why `T`?** Same pattern as `filterEvents` in #2. `T` is a placeholder type filled in at each call. `extends DatedEvent` means it must have at least `date` and `time`. Pass in `TechEvent[]` and you get `TechEvent[]` back, so `upcoming[0].technology` still type-checks. With `(events: DatedEvent[]): DatedEvent[]` the result would only know `date` and `time`. The name `T` is just a convention (for "Type"); `<EventType extends DatedEvent>` means the same thing.

**The default `today` parameter.** If the caller doesn't pass `today`, it's computed when the function is called:

```ts
new Date()                    // now
  .toISOString()              // "2026-11-07T22:14:00.000Z"
  .slice(0, 10)               // "2026-11-07"  (first 10 characters: YYYY-MM-DD)
```

Having `today` as a parameter also makes the function easy to test: pass a fixed date instead of depending on the real clock.

**`.filter()` compares dates as strings.** `"2026-11-05" >= "2026-11-07"` is a plain string comparison, character by character. It works because `YYYY-MM-DD` is zero-padded and goes from largest unit to smallest, so alphabetical order is the same as date order. It would **not** work for `"7/11/2026"` or `"2026-11-7"`.

```ts
const today = "2026-11-07";
"2026-11-05" >= today // false: removed
"2026-11-07" >= today // true: kept (today counts as upcoming)
"2026-11-10" >= today // true: kept
```

**`.sort()` with `localeCompare`.** The comparator gets two events, `a` and `b`. A template literal joins date and time into one sortable string, and `localeCompare` returns a negative number (`a` first), `0` (same position) or a positive number (`b` first).

```ts
`${a.date} ${a.time}` // "2026-11-10 09:00"
`${b.date} ${b.time}` // "2026-11-07 14:00"
"2026-11-10 09:00".localeCompare("2026-11-07 14:00") // positive: b comes first
```

Joining date **and** time matters: two events on the same day are then ordered by time (`09:00` before `18:00`). Like the date, `HH:mm` must be zero-padded and 24-hour (`"9:00"` would sort after `"18:00"`).

**`.filter()` before `.sort()` protects the input.** `.sort()` changes the array it's called on. Here it runs on the **new** array that `.filter()` returned, so the caller's `events` array is never changed. Sorting `events` directly would reorder the caller's data. (`.toSorted()` is the newer non-mutating alternative.)

**Two limitations to know:**

- **`toISOString()` is UTC, not local time.** At 10 PM on Nov 7 in New York it's already Nov 8 in UTC, so the default `today` would be `"2026-11-08"` and today's evening events would disappear. A local date needs something like `new Date().toLocaleDateString("en-CA")`, which gives `YYYY-MM-DD` in the user's time zone.
- **It filters by date only.** An event at 9 AM today still shows at 6 PM today. That's fine if "upcoming" means "today or later". If it means "not started yet", compare the full `` `${date} ${time}` `` string against the current date and time.

**Interview Q:** *Why can you compare dates as strings here?* Because they're ISO `YYYY-MM-DD`: fixed width, zero-padded, biggest unit first. For that format, string order equals chronological order. Any other format should be parsed into a `Date` (or a timestamp) first.

**Interview Q:** *What does a `.sort()` comparator return, and does `.sort()` mutate?* Negative to put `a` first, positive to put `b` first, `0` to keep their order. `.sort()` mutates the array in place and also returns it, so sort a copy (`.filter()`, `[...arr]` or `.toSorted()`) when the input belongs to someone else.

**Interview Q:** *What's the risk with `new Date().toISOString().slice(0, 10)` as "today"?* It's the UTC date. Near midnight it can be a day ahead of or behind the user's local date.

---

## Filters in the URL

### 5. Filter pills as links: `URLSearchParams`, `?mode=` and `aria-current`

`ModeFilter` shows four pills: **All**, **Online**, **In person**, **Hybrid**. The key idea: **the component doesn't filter anything.** Each pill is a `<Link>` to a URL with `?mode=` set. The page reads `searchParams` and runs `filterEvents` (#3). Because the choice lives in the URL, it survives a refresh, works with the back button, can be shared, and needs no `useState`, so the component can stay a Server Component. (The general idea is in [Talent Pipeline notes #39](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#39-the-url-as-state).)

```
click "Online" → /?q=React&mode=online#events → page reads searchParams → filterEvents → online events shown
```

**The options list.** Each option has a required `label` and an optional `value`. **All** has no `value` on purpose: "All" means "no mode filter", not a fourth mode.

```tsx
const OPTIONS: { value?: EventMode; label: string }[] = [
  { label: "All" },                                         // no value = no ?mode=
  { value: "online", label: FORMAT_LABELS.online },
  { value: "in-person", label: FORMAT_LABELS["in-person"] }, // brackets: "in-person" has a hyphen
  { value: "hybrid", label: FORMAT_LABELS.hybrid },
];
```

`FORMAT_LABELS.in-person` would be read as `FORMAT_LABELS.in - person` (a subtraction), so a key with a hyphen needs bracket notation. Taking the labels from `FORMAT_LABELS` keeps the pill text the same as the badge on each event card.

**The props.** The component needs the current search (`q`, `where`) plus the current `mode`. `&` is an intersection type: it combines both types into one ([Talent Pipeline notes #9](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#9-intersection-types-)).

```tsx
type ModeFilterProps = EventSearch & { mode?: EventMode };
// same as { q?: string; where?: string; mode?: "online" | "in-person" | "hybrid" }

const ModeFilter = ({ q, where, mode }: ModeFilterProps) => ( /* ... */ );
```

**Building each link with `URLSearchParams`.** For every option, start from an empty set of params and add only what has a value. This **keeps the other filters**: clicking a pill changes `mode` but keeps the user's `q` and `where`.

```tsx
const params = new URLSearchParams();
if (q) params.set("q", q);
if (where) params.set("where", where);
if (value) params.set("mode", value);   // "All" has no value, so ?mode= is dropped
const query = params.toString();        // "q=React&where=New+York&mode=online"
```

`URLSearchParams` also encodes values for you: the space in `New York` becomes `+`, and `&` or `#` inside a value can't break the URL. That's safer than building `"?q=" + q` by hand ([Talent Pipeline notes #17](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#17-urlsearchparams-and-encodeuricomponent)).

**Which pill is selected.** Compare the option's value with the current mode:

```tsx
const selected = value === mode;
```

With no `?mode=` in the URL, `mode` is `undefined`, and **All**'s `value` is also `undefined`. `undefined === undefined` is `true`, so **All** is selected by default without any special case.

**The link itself.** A ternary inside the template literal adds `?` only when there are params, so the "All with no search" link is a clean `/#events` rather than `/?#events`.

```tsx
<Link
  key={label}                                       // stable, unique key for .map()
  href={`/${query ? `?${query}` : ""}#events`}      // "/?mode=online#events" or "/#events"
  scroll={false}                                    // don't jump to the top of the page
  aria-current={selected ? "true" : undefined}      // screen readers hear "current"
  className={`... rounded-full border px-4 ${
    selected ? "border-accent bg-accent text-on-accent" : "border-border text-text"
  }`}
>
  {label}
</Link>
```

| Piece | Why |
|---|---|
| `#events` | The fragment points at `<section id="events">`, so the user stays at the results list |
| `scroll={false}` | Stops Next.js from scrolling to the top after the navigation |
| `aria-current="true"` | Tells assistive tech which pill is active. `undefined` leaves the attribute out entirely. `"page"` is meant for links to the current page, so `"true"` fits a filter better |
| `key={label}` | React needs a stable key for items made by `.map()`; the labels are unique |
| conditional `className` | Shared classes for every pill, plus one of two color sets depending on `selected` |
| `<nav aria-label="...">` | Groups the pills as a named navigation region |

**Planned fix: validate `?mode=` on the page.** `EventMode` is only a TypeScript type, and types disappear at runtime. A user can type `?mode=banana` into the URL. The page should check the value before trusting it:

```ts
const MODES = ["online", "in-person", "hybrid"] as const;
const isEventMode = (value?: string): value is EventMode =>
  MODES.includes(value as EventMode);

const raw = firstParam(searchParams.mode);       // from #3
const mode = isEventMode(raw) ? raw : undefined; // anything unknown means "All"
```

**Interview Q:** *Why make filters links instead of buttons with `useState`?* The URL becomes the single source of truth. Filters survive a refresh, the back button works, and the link can be shared. The server can read `searchParams` and filter before rendering, and the component needs no client JavaScript.

**Interview Q:** *Why build the query with `URLSearchParams` instead of string concatenation?* It encodes special characters (spaces, `&`, `#`, `=`) correctly, and it's easy to add only the params that have values, so the other filters are kept.

**Interview Q:** *TypeScript says `mode` is `EventMode`. Why validate it?* Types are checked at compile time only. URL input arrives at runtime and can be any string, so it must be checked (for example with a type guard) before use.

---

## Next.js rendering and config

### 6. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"

This was a real production build failure, and it only appeared in `next build`.

```text
Error: Route "/events/[slug]": Uncached data was accessed outside of <Suspense>.
```

With `cacheComponents: true`, Next.js prerenders a static shell of each page. Anything only known at request time, such as a `fetch` or `await params`, must be cached with `"use cache"` or sit inside `<Suspense>`, so the shell can be sent first.

**Before: the failing code.** The whole page awaits request-time data.

```tsx
// app/events/[slug]/page.tsx
const EventDetailsPage = async ({ params }: EventPageProps) => {
  const { slug } = await params;
  const res = await fetch(`${BASE_URL}/api/events/${slug}`);
  // ...
};
```

**After: the fix.** The page is synchronous, and the data work moves inside the boundary.

1. The page is no longer `async`, so it waits for nothing and Next.js can send it immediately. `params` is passed along still unresolved, as a Promise.
2. `<Suspense>` sets the loading state, so visitors see "Loading event..." until the content is ready.
3. `EventDetailsContent` does the slow work: it reads the slug and fetches the event. When it finishes, its content replaces the loading message.

```tsx
const EventDetailsPage = ({ params }: EventPageProps) => (
  <Suspense fallback={<div>Loading event...</div>}>
    <EventDetailsContent params={params} /> {/* awaits params and fetches here */}
  </Suspense>
);
```

**Trade-off I found by testing:** a missing event now shows the 404 page, but the response status is 200. The status is sent with the static shell, before `notFound()` runs during streaming.

**Why it's worth having anyway:** without `Suspense`, visitors see nothing until all the data is ready. With it, the navbar and the fallback appear right away, and only the part inside the boundary waits. This applies in production too, because `next.config.ts` is used for both development and production, and the error stopped `npm run build`, so the site couldn't have been deployed.

**Interview Q:** *Why use `<Suspense>` in a Server Component page?* To send the rest of the page immediately and stream the slow part in when it's ready. The `fallback` is what shows in the meantime, so one slow query doesn't hold up the whole page.

**Interview Q:** *What is Partial Prerendering?* The page's static parts are prerendered and sent immediately, then dynamic parts stream in through `<Suspense>` boundaries. Users see content fast, even when some data is slow.

### 7. Using `rewrites` as a reverse proxy for analytics

Requests go to my own domain and Next.js forwards them. Ad blockers often block known analytics domains, so a same-origin path makes the tracking more reliable.

- A **redirect** tells the browser to go to a new URL, and the address changes.
- A **rewrite** happens on the server, and the browser's URL stays the same.

**Forward the requests.** Anything sent to `/ingest` goes on to PostHog.

```ts
// next.config.ts
async rewrites() {
  return [
    { source: '/ingest/static/:path*', destination: 'https://us-assets.i.posthog.com/static/:path*' },
    { source: '/ingest/:path*', destination: 'https://us.i.posthog.com/:path*' },
  ];
},
skipTrailingSlashRedirect: true, // required for PostHog's trailing-slash API requests
```

**Point PostHog at my own path.** `api_host` sends its requests to `/ingest` instead of posthog.com.

```ts
// instrumentation-client.ts
posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
  api_host: '/ingest', // same-origin path, not posthog.com
});
```

**Interview Q:** *What's the difference between a redirect and a rewrite?* A redirect sends the browser to a new URL with a 3xx response. A rewrite serves the content from another location while keeping the original URL.

---

## React hooks

### 8. `useDropdownMenu`: click outside, focus and arrow keys in a custom hook

The account menu's behavior lives in a [custom hook](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#18-custom-hooks), and the component keeps only the markup. The hook file has no `"use client"` of its own, because it's only imported by a component that does.

**The state and the refs.** `isOpen` changes what's on screen, so it's state. The three refs only point at DOM elements, and the render output doesn't depend on them, so they don't need to trigger a re-render ([`useRef` vs `useState`](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#22-useref-vs-usestate)).

```ts
// hooks/useDropdownMenu.ts
const [isOpen, setIsOpen] = useState(false);
const containerRef = useRef<HTMLSpanElement>(null); // wraps the button and the menu
const buttonRef = useRef<HTMLButtonElement>(null);  // gets focus back on Escape
const menuRef = useRef<HTMLDivElement>(null);       // used to find the menu items
```

**Listen only while the menu is open.** With `[isOpen]` as the dependency, the effect runs each time the menu opens or closes. The early return means a closed menu adds no listeners to the page.

```ts
useEffect(() => {
  if (!isOpen) return;
  // focus and listeners go here
}, [isOpen]);
```

**Moving focus when it opens.** Effects run after React has updated the DOM, so `menuRef.current` already points at the menu here. `querySelectorAll` returns a `NodeList`, which has no `.indexOf()`, so the spread makes it a real array. `items` is a function so each call reads the current items.

```ts
const items = () => [...(menuRef.current?.querySelectorAll<HTMLElement>('[role="menuitem"]') ?? [])];
items()[0]?.focus();
```

**Clicking outside.** `.contains()` is true when the clicked element is the container or anything inside it. The `as Node` is needed because `e.target` is typed as the broader `EventTarget`.

```ts
const handlePointerDown = (e: PointerEvent) => {
  if (!containerRef.current?.contains(e.target as Node)) setIsOpen(false);
};
```

The container wraps the **button as well as the menu**, and that matters. If only the menu counted as inside, pressing the open button would close the menu on `pointerdown`, then the `click` that follows would toggle it open again, so it could never be closed that way.

**Escape.** The menu closes and focus goes back to the button. Otherwise the focused item is removed from the page and a keyboard user loses their place.

```ts
if (e.key === "Escape") {
  setIsOpen(false);
  buttonRef.current?.focus();
}
```

**Arrow keys that wrap around.** `preventDefault` stops the page from scrolling. The last line finds the next position and wraps at both ends.

```ts
if (e.key === "ArrowDown" || e.key === "ArrowUp") {
  e.preventDefault();
  const all = items();
  const step = e.key === "ArrowDown" ? 1 : -1;
  const current = all.indexOf(document.activeElement as HTMLElement);
  all[(current + step + all.length) % all.length]?.focus();
}
```

Adding `all.length` matters because JavaScript's `%` keeps the sign of the left side, so `-1 % 3` is `-1`, not `2`. With 3 items:

| Focus is on | Key | Calculation | Moves to |
|---|---|---|---|
| Index 2 (last) | ArrowDown | `(2 + 1 + 3) % 3` | 0 (first) |
| Index 0 (first) | ArrowUp | `(0 - 1 + 3) % 3` | 2 (last) |

**Known edge case:** if focus isn't on an item, `indexOf` returns `-1`. ArrowDown still lands on the first item, but ArrowUp gives `(-1 - 1 + 3) % 3 = 1`, the second-to-last item rather than the last.

**Registering and removing the listeners.** The cleanup runs when the menu closes and when the component unmounts ([effect cleanup](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#20-useeffect-cleanup-and-dependency-arrays)). `removeEventListener` only works when given the same function that was added, which is why the handlers are named variables instead of inline arrow functions.

```ts
document.addEventListener("pointerdown", handlePointerDown);
document.addEventListener("keydown", handleKeyDown);
return () => {
  document.removeEventListener("pointerdown", handlePointerDown);
  document.removeEventListener("keydown", handleKeyDown);
};
```

**What the hook hands back.** The component gets values and actions, and never touches `setIsOpen`. `toggle` uses a [functional update](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#19-usestate-and-functional-updates) so it always flips the latest value.

```ts
return {
  isOpen,
  toggle: () => setIsOpen((open) => !open),
  close: () => setIsOpen(false),
  containerRef,
  buttonRef,
  menuRef,
};
```

**Interview Q:** *How do you detect a click outside an element in React?* Put a ref on the element, listen on `document`, and check `ref.current.contains(e.target)`. Add the listener in an effect and remove it in the cleanup.

**Interview Q:** *Why store a DOM element in `useRef` instead of `useState`?* Changing a ref doesn't cause a re-render, and the element is only needed to call methods like `.focus()`. Keeping it in state would add renders for no visible change.

**Interview Q:** *Why can you call `.focus()` on the menu inside `useEffect`?* The effect runs after React has committed the update to the DOM, so the menu exists and the ref is attached. During render it wouldn't exist yet.

**Interview Q:** *Why doesn't `removeEventListener(() => ...)` work with a new arrow function?* It matches by function reference. A new arrow function is a different object, so nothing gets removed and the original listener stays active.

**Interview Q:** *How do you wrap an index in both directions?* Use `(index + step + length) % length`. Adding `length` first keeps the value positive, because `%` in JavaScript can return a negative number.

**Interview Q:** *What keyboard behavior does an accessible dropdown menu need?* Focus moves into the menu when it opens, arrow keys move between items, and Escape closes it and returns focus to the button that opened it. `aria-expanded` and `aria-haspopup` tell screen readers its state.

---

## React components

### 9. `AuthPanel`: one form for two tabs, a typed input wrapper, and an auth check first

Sign in and Create account are one form. The component only renders: the tab state and submit logic come from `useAuthForm` ([auth notes #7](../auth/auth-from-StackMeet.md)).

**One piece of state drives the screen.** Only `mode` is stored. `isSignUp` and `copy` are worked out from it on each render, so they can't get out of sync ([derived values](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#24-derived-values-instead-of-state)).

```tsx
// components/auth/AuthPanel.tsx
const { mode, setMode, isSubmitting, handleSubmit } = useAuthForm(initialMode);
const isSignUp = mode === "sign-up";
const copy = COPY[mode];
```

**A lookup object instead of many ternaries.** Each tab's text sits together, and the JSX just reads `copy.heading`. If `COPY` were missing a key that `AuthMode` allows, `COPY[mode]` would be a compile error.

```tsx
const COPY = {
  "sign-in": { heading: "Welcome back", cta: "Sign in", switchLink: "Create one" },
  "sign-up": { heading: "Create your account", cta: "Create account", switchLink: "Sign in" },
};
```

**Rendering the tabs from an array.** The type on `TABS` means each `mode` must be a valid `AuthMode`. `aria-selected` tells screen readers which tab is active, because color alone doesn't.

```tsx
const TABS: { mode: AuthMode; label: string }[] = [
  { mode: "sign-in", label: "Sign in" },
  { mode: "sign-up", label: "Create account" },
];

{TABS.map((tab) => (
  <button key={tab.mode} type="button" role="tab" aria-selected={tab.mode === mode} onClick={() => setMode(tab.mode)}>
    {tab.label}
  </button>
))}
```

`type="button"` matters for any button inside a form, because a button with no type defaults to `submit`. These tabs are outside the form, so here it's just a safe habit. **Known limitation:** a full ARIA tabs pattern also has `role="tabpanel"` and arrow-key navigation between tabs. This follows the design's markup, which only has the roles above.

**Why typed text survives switching tabs.** The name field is added and removed, but the email input keeps what the user typed. React matches elements by their position among siblings. When the condition is false, `false` still fills that position, so the email field stays in the same slot and React reuses the same DOM input.

```tsx
<form onSubmit={handleSubmit} noValidate>
  {isSignUp && <AuthField id="auth-name" name="name" label="name" type="text" required />}
  <AuthField id="auth-email" name="email" label="email" type="email" required />
</form>
```

`noValidate` turns off the browser's own error bubbles. The server still validates everything.

**One password input, two meanings.** The `autoComplete` value tells password managers whether to fill a saved password or suggest a new one.

```tsx
autoComplete={isSignUp ? "new-password" : "current-password"}
```

**A typed wrapper around `<input>`.** `ComponentProps<"input">` gives the wrapper every real input prop, such as `type`, `name` and `placeholder`, without listing them. The intersection adds `label` and `hint`, and makes `id` required so the label can always connect to its input. The rest pattern takes out what the wrapper uses, and the spread passes everything else on.

```tsx
// components/auth/AuthField.tsx
type AuthFieldProps = ComponentProps<"input"> & {
  id: string;
  label: string;
  hint?: string;
};

const AuthField = ({ id, label, hint, ...inputProps }: AuthFieldProps) => (
  <div>
    <label htmlFor={id}>{label}</label>
    <input id={id} className="..." {...inputProps} />
    {hint && <span>{hint}</span>}
  </div>
);
```

Order matters with a spread: later props win. Because `{...inputProps}` comes after `className`, a caller's `className` would replace the wrapper's styles entirely.

**Checking auth before the panel renders.** My first version put the auth check beside the panel, so a signed-in user could briefly see the form before being redirected. Putting the check at the top of the page fixed that but failed `npm run build`, because `auth()` reads cookies and `cacheComponents` requires that to be inside `<Suspense>` (see #6). The working version puts the check and the panel in one async component, inside the boundary.

```tsx
// app/auth/sign-in/page.tsx
async function SignInContent({ searchParams }: SignInPageProps) {
  const session = await auth();
  if (session?.user) redirect("/");
  const initialMode = parseAuthMode(await searchParams);
  return <AuthPanel key={initialMode} initialMode={initialMode} />;
}

export default function SignInPage({ searchParams }: SignInPageProps) {
  return (
    <div data-full-width className="...">
      <Suspense fallback={null}>
        <SignInContent searchParams={searchParams} />
      </Suspense>
    </div>
  );
}
```

The type check and lint both passed on the broken version. Only the build prerenders pages, so only the build can catch it.

**Keeping an old URL working.** Creating an account used to be its own page. Now `/auth/sign-up` only redirects, so old links still work. `redirect()` from a Server Component sends a 307 temporary redirect, and because this page reads nothing per visitor, it's built as a static route.

```tsx
// app/auth/sign-up/page.tsx
export default function SignUpPage() {
  redirect("/auth/sign-in?tab=create");
}
```

**Opening a tab from the URL.** The parsing is a plain function in `lib/`, not in the page. `searchParams` values are whatever someone typed, so only the exact value `create` opens that tab, and anything else falls back to Sign in. `firstParam` handles a repeated `?tab=a&tab=b` (#3).

```ts
// lib/auth-query.ts
export const parseAuthMode = (params: AuthSearchParams): AuthMode =>
  firstParam(params.tab) === "create" ? "sign-up" : "sign-in";
```

The hook uses that value as the starting state.

```ts
// hooks/useAuthForm.ts
export function useAuthForm(initialMode: AuthMode = "sign-in") {
  const [mode, setMode] = useState<AuthMode>(initialMode);
```

**The URL sets the first tab only.** After that, clicking a tab calls `setMode` and the URL stays the same. That's a deliberate difference from the [URL as state](./typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#39-the-url-as-state) pattern used by the home filters, where the URL is the single source of truth. Which tab is showing doesn't need to survive a refresh or be shared, so plain state is enough.

**Why the component needs a `key`.** `useState` reads its starting value only the first time a component mounts. Going from `?tab=create` to `/auth/sign-in` through the navbar changes the prop, but React keeps the same component and its old state, so the Create account tab would stay open. A different `key` tells React it's a new component, so the state starts over.

```tsx
<AuthPanel key={initialMode} initialMode={initialMode} />
```

**Interview Q:** *Why does `{condition && <Field />}` keep the next input's value when it toggles?* React matches children by position. `false` holds that position, so the following element doesn't move and React keeps its DOM node and state.

**Interview Q:** *How would you force an input to reset when something changes?* Give it a different `key`. A new key tells React it's a different element, so it creates a fresh one.

**Interview Q:** *How do you type a component that wraps a native element?* Use `ComponentProps<"input">`, add your own props with `&`, and pass the rest through with a spread.

**Interview Q:** *Does the order of `className` and `{...props}` matter?* Yes. Later props override earlier ones, so the spread's position decides whether the caller can replace a value.

**Interview Q:** *Why isn't `aria-selected` just decoration?* It exposes the state to assistive technology. Without it, a screen reader user can't tell which tab is active from a color change.

**Interview Q:** *How do you stop signed-in users from seeing a sign-in form?* Check the session on the server before rendering the form, and redirect if there is one. With `cacheComponents`, that check has to be inside `<Suspense>`.

**Interview Q:** *Why doesn't `useState(prop)` update when the prop changes?* The argument is only used on the first render. Later renders ignore it. To restart the state when the prop changes, give the component a `key` based on that prop.

**Interview Q:** *When should UI state live in the URL instead of `useState`?* Use the URL when it should survive a refresh, work with the back button, or be shareable, like search filters. Use `useState` for short-lived details like which tab of a form is open.

**Interview Q:** *Why validate a value read from `searchParams`?* It comes from the URL, so it can be anything, and TypeScript types don't exist at runtime. Check it against the values you allow and fall back to a default.

**Interview Q:** *How do you retire a page without breaking old links?* Keep the route and redirect it to the new location. Use a temporary redirect (307) if it may change, or a permanent one (308) when it won't.
