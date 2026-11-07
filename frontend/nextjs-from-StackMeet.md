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
- [Next.js rendering and config](#nextjs-rendering-and-config)
  5. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"
  6. Using `rewrites` as a reverse proxy for analytics

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

## Next.js rendering and config

### 5. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"

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

### 6. Using `rewrites` as a reverse proxy for analytics

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
