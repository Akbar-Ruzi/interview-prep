# Interview Notes: Next.js Rendering and Config

> Every snippet is from my project (StackMeet, a developer events app: Next.js 16, React 19, TypeScript, Mongoose 8, Auth.js v5), shortened where needed.
> Snippets marked **Planned fix** show the fix I would make. They aren't in the project yet.
> For each concept: what it is, the project code, and a likely interview question.
> Other notes from this project: [Next.js API and server code](../backend/nextjs-api-from-StackMeet.md), [Authentication and authorization](../auth/auth-from-StackMeet.md) and [MongoDB and Mongoose](../databases/mongodb-mongoose-from-StackMeet.md).

## Contents

- [TypeScript](#typescript)
  1. The `satisfies` operator
- [Next.js rendering and config](#nextjs-rendering-and-config)
  2. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"
  3. Using `rewrites` as a reverse proxy for analytics

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

---

## Next.js rendering and config

### 2. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"

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

### 3. Using `rewrites` as a reverse proxy for analytics

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
