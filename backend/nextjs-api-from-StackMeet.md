# Interview Notes: Next.js API and Server Code

> Every snippet is from my project (StackMeet, a developer events app: Next.js 16, React 19, TypeScript, Mongoose 8, Auth.js v5), shortened where needed.
> Snippets marked **Planned fix** show the fix I would make. They aren't in the project yet.
> For each concept: what it is, the project code, and a likely interview question.
> Other notes from this project: [Next.js rendering and config](../frontend/nextjs-from-StackMeet.md), [Authentication and authorization](../auth/auth-from-StackMeet.md) and [MongoDB and Mongoose](../databases/mongodb-mongoose-from-StackMeet.md).

## Contents

- [Next.js API and server code](#nextjs-api-and-server-code)
  1. Route Handlers and HTTP status codes
  2. Server Actions and returning a result object
  3. Uploading a file: wrapping a callback API in a Promise
  4. Constants in a `'use server'` file
  5. Reading a request body: `await req.json()` and `as`
  6. A full route, step by step: sign-up
  7. `import "server-only"` and the data access layer
  8. Loading page data: failure handling per call, `notFound()` and why reads aren't Server Actions
  9. Three ways to reach the database: Route Handlers, Server Actions and data functions

---

## Next.js API and server code

### 1. Route Handlers and HTTP status codes

A `route.ts` file creates an API endpoint, and each exported function handles one HTTP method.

```ts
// app/api/events/route.ts
export async function POST(req: NextRequest) { /* create */ }
export async function GET() { /* list */ }

// app/api/events/[slug]/route.ts
export async function GET(request: NextRequest, { params }: RouteParams) { /* read one */ }
export async function PATCH(request: NextRequest, { params }: RouteParams) { /* update */ }
export async function DELETE(request: NextRequest, { params }: RouteParams) { /* delete */ }
```

| Code | Meaning | Where it's used in StackMeet |
|---|---|---|
| 200 | OK | Fetching or updating an event |
| 201 | Created | Event or account created |
| 400 | Invalid input | Missing fields, invalid email |
| 401 | Not signed in | Creating an event while signed out |
| 403 | Not allowed | Someone else's event, or over the limit |
| 404 | Not found | Unknown slug |
| 409 | Conflict | Email or title already exists |
| 500 | Server error | Unexpected failures |

**Interview Q:** *What's the difference between PUT and PATCH?* PUT replaces the whole resource, while PATCH changes only the fields you send.

### 2. Server Actions and returning a result object

A Server Action is a server function that a client component can call directly, without you writing a fetch or an API route.

**The server side.** `'use server'` marks the file's exports as Server Actions. A Server Action is a public POST endpoint, so it needs the same validation and permission checks as an API route. Client-side validation alone isn't enough.

The function returns a result object instead of using `throw`, because expected failures, like a duplicate booking, are normal outcomes the UI can display. In production, Next.js also hides the message of errors thrown from Server Actions.

```ts
// lib/actions/booking.action.ts
'use server';

export const createBooking = async ({ eventId, email }: CreateBookingParams) => {
  await connectDB();
  // ...
  return { success: true } satisfies CreateBookingResult;
};
```

**The client side.** The component calls the action like a normal async function, then checks `result.success` to choose what to show.

```ts
// components/events/BookEvent.tsx (client component)
const result = await createBooking({ eventId, slug, email: normalizedEmail });
if (result.success) {
  setSubmitted(true);
} else {
  setErrorMessage(result.message ?? 'Booking creation failed. Please try again.');
}
```

**Interview Q:** *Server Action or Route Handler?* Server Actions suit mutations from your own UI. Route Handlers suit endpoints that outside clients, webhooks or other apps need to call.

### 3. Uploading a file: wrapping a callback API in a Promise

**Read the file from the form.** Files use `FormData` because JSON can't carry binary data without encoding it. The `instanceof File` check rejects a request with no real file.

```ts
// app/api/events/route.ts
const formData = await req.formData();
const imageValue = formData.get('image');
if (!(imageValue instanceof File)) {
  return NextResponse.json({ message: 'Image file is required' }, { status: 400 });
}
```

**Turn the file into a `Buffer`.** This holds the whole file in memory, so the type and size should be checked before this point. My code doesn't do that yet, and it's in my plan.

```ts
const buffer = Buffer.from(await imageValue.arrayBuffer());
```

**Wrap the upload in a Promise.** Cloudinary's upload stream reports its result through a callback, so wrapping it in a Promise allows `await`. The callback calls `reject` on a failure and `resolve` on success. The database later stores only the image's URL, not the file itself.

```ts
const uploadResult = await new Promise<CloudinaryUploadResult>((resolve, reject) => {
  cloudinary.uploader
    .upload_stream({ resource_type: 'image', folder: 'StackMeet' }, (error, result) => {
      if (error) return reject(error);
      if (!result?.secure_url) return reject(new Error('Cloudinary upload failed'));
      resolve({ secure_url: result.secure_url });
    })
    .end(buffer);
});
```

**Interview Q:** *How do you turn a callback API into a Promise?* Wrap it in `new Promise`, call `reject` on an error and `resolve` on success. `util.promisify` does this for standard Node-style callbacks.

### 4. Constants in a `'use server'` file

These constants are used by `getValidationError` in the same file. Naming them keeps the rules in one place and explains what a bare `50` or `15` means.

```ts
// lib/actions/booking.action.ts
const MAX_NAME_LENGTH = 50;
const MIN_PHONE_DIGITS = 7;
const MAX_PHONE_DIGITS = 15;
const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const PHONE_REGEX = /^\+?[\d\s().-]+$/;
```

They can't be exported from this file, because a `'use server'` file can only export async functions. **Planned fix:** when the new booking form needs the same rules, move them into a small shared file that both the form and the server import.

**Interview Q:** *Why can a `'use server'` file only export async functions?* Each export becomes a function the browser can call over the network, and a network call is always asynchronous. A plain value like a number or regex can't be called that way.

**Interview Q:** *Why validate on both the client and the server?* Client validation gives fast feedback, but it's only for convenience, because anyone can skip it and call the server directly. Server validation is what actually protects the data.

### 5. Reading a request body: `await req.json()` and `as`

This reads the JSON that the client sent and tells TypeScript what shape to expect.

**Describe the expected shape.** Every field is optional because a client can leave any of them out. That makes TypeScript require handling for missing values.

```ts
// app/api/auth/signup/route.ts
type SignupBody = {
  name?: string;
  email?: string;
  password?: string;
  phone?: string;
};
```

**Read the body.**

- `req.json()` reads the request body and parses it as JSON. It returns a Promise because the body arrives as a stream, so it needs `await`.
- The parentheses make the `await` finish first, then `as SignupBody` applies to the parsed result.
- `as` is a type assertion. It tells TypeScript to treat the value as this type, and it's removed at compile time, so nothing is checked at runtime.
- If the body isn't valid JSON, `req.json()` throws. In this route the `catch` block turns that into a 500, even though it's really the client's mistake and a 400 would be more accurate.

```ts
const body = (await req.json()) as SignupBody;
```

**Handle missing values.** Because the fields are optional, the code uses `?.` and `??`.

`as` doesn't protect against wrong types. If a client sends `{ "name": 123 }`, `body.name?.trim()` throws at runtime. The real checks are the validation that follows, or a schema library like Zod.

```ts
const name = body.name?.trim() ?? "";
const email = body.email?.trim().toLowerCase() ?? "";
const password = body.password ?? "";
```

**Interview Q:** *Does `as SignupBody` validate the request body?* No. It only changes what TypeScript believes about the value, and it's gone from the compiled JavaScript. Data from a request must be validated at runtime.

**Interview Q:** *What's the difference between `as` and a type annotation?* An annotation, such as `const body: SignupBody = value`, makes the compiler check that the value matches the type. `as` tells the compiler to trust you, so a wrong assertion can hide a real bug. `req.json()` returns `Promise<any>`, so an annotation wouldn't check anything here either.

### 6. A full route, step by step: sign-up

**Why the file is called `route.ts`.** Next.js uses file names to decide what a file does, and it only treats certain names as special. `page.tsx` shows a page, while `route.ts` creates an API endpoint. The folders give the URL, and the exported function names give the HTTP methods.

```text
app/api/auth/signup/route.ts  +  export POST  =  POST /api/auth/signup
```

**Imports.** These bring in the Next.js request and response helpers, bcrypt for hashing, the database connection and the `User` model.

```ts
// app/api/auth/signup/route.ts
import { NextRequest, NextResponse } from "next/server";
import bcrypt from "bcryptjs";

import connectDB from "@/lib/mongodb";
import User from "@/database/user.model";
```

**Request type and email pattern.** `SignupBody` describes the expected input (see section 5). `EMAIL_REGEX` checks for the basic `text@text.text` format.

```ts
type SignupBody = {
  name?: string;
  email?: string;
  password?: string;
  phone?: string;
};

const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
```

**The handler's outline.** Exporting a function named `POST` handles POST requests. Steps 1 to 7 below all run inside this `try`. Each failed check returns immediately, which keeps the success path unnested.

```ts
export async function POST(req: NextRequest) {
  try {
    // Steps 1 to 7
  } catch (error) {
    // Step 8
  }
}
```

**Step 1: Read and clean the input.** `trim` and `toLowerCase` make `Sam@Example.com ` and `sam@example.com` the same account. The password isn't trimmed, because spaces can be part of it.

For the phone, `||` turns an empty string into `undefined`, so a blank phone isn't saved. `??` wouldn't do this, because it only replaces `null` and `undefined`.

```ts
const body = (await req.json()) as SignupBody;

const name = body.name?.trim() ?? "";
const email = body.email?.trim().toLowerCase() ?? "";
const password = body.password ?? "";
const phone = body.phone?.trim() || undefined;
```

**Step 2: Validate, and stop early with a 400.** Validation runs before `connectDB()` and bcrypt, so bad input doesn't cost a database call or a slow hash.

First, check that the required fields are present:

```ts
if (!name || !email || !password) {
  return NextResponse.json(
    { message: "Name, email and password are required" },
    { status: 400 }
  );
}
```

Next, check the name's length:

```ts
if (name.length < 2 || name.length > 50) {
  return NextResponse.json(
    { message: "Name must be between 2 and 50 characters" },
    { status: 400 }
  );
}
```

Then check the email's format:

```ts
if (!EMAIL_REGEX.test(email)) {
  return NextResponse.json(
    { message: "Please provide a valid email address." },
    { status: 400 }
  );
}
```

Finally, check the password's length. There are two separate checks: a minimum of 8 characters, and a maximum of 64 characters and 72 bytes. bcrypt only uses the first 72 bytes, so the byte check is still needed for passwords with emoji or accented letters (see [Hashing passwords with bcrypt](../auth/auth-from-StackMeet.md)).

```ts
if (password.length < 8) {
  return NextResponse.json(
    { message: "Password must be at least 8 characters" },
    { status: 400 }
  );
}

// bcrypt only uses the first 72 bytes, and characters like emoji take more than one byte,
// so a password within the character limit can still be too long for bcrypt
if (password.length > 64 || Buffer.byteLength(password, "utf8") > 72) {
  return NextResponse.json(
    { message: "Password is too long. Please use a shorter one." },
    { status: 400 }
  );
}
```

**Step 3: Connect to the database.** This runs only after the input has passed validation.

```ts
await connectDB();
```

**Step 4: Reject an email that's already registered.** A duplicate returns a 409 Conflict. **Known issue:** the check-then-create steps can race under simultaneous sign-ups (see [A unique index beats "check, then insert"](../databases/mongodb-mongoose-from-StackMeet.md)).

```ts
const existingUser = await User.findOne({ email }).lean();
if (existingUser) {
  return NextResponse.json(
    { message: 'Email already exists' },
    { status: 409 }
  );
}
```

**Step 5: Hash the password.** This means plain text is never stored.

```ts
const hashedPassword = await bcrypt.hash(password, 12);
```

**Step 6: Create the user.** `role: 'user'` is hard-coded and never comes from the request, so nobody can sign themselves up as an admin.

```ts
const createdUser = await User.create({
  name,
  email,
  password: hashedPassword,
  phone, // optional
  role: 'user',
});
```

**Step 7: Return the new user, without the password.** The response lists each field by hand, so the password hash can't leak. The 201 status means the account was created.

```ts
return NextResponse.json(
  {
    message: 'Account created successfully',
    user: {
      id: createdUser._id.toString(),
      name: createdUser.name,
      email: createdUser.email,
      phone: createdUser.phone ?? null,
      role: createdUser.role,
    }
  },
  { status: 201 }
);
```

**Step 8: Turn anything unexpected into a 500.** The `catch` block logs the real error on the server and sends the client a general message.

```ts
console.error("Signup failed:", error);
return NextResponse.json(
  { message: "Signup failed. Please try again later." },
  { status: 500 }
);
```

**Interview Q:** *How does Next.js know a file is an API endpoint?* From the file name. `route.ts` inside `app/` handles requests at the URL made from its folder path, and each exported function named after an HTTP method, such as `POST`, handles that method. A method with no export gets a 405.

**Interview Q:** *Why don't you return the created user directly from `User.create`?* It includes the password hash and internal fields. Listing the fields you want means a new sensitive field on the model can't leak by accident.

**Interview Q:** *Why hard-code `role: 'user'` instead of reading it from the body?* Clients control the request body, so accepting a role would let anyone make themselves an admin. This is called a mass assignment vulnerability.

### 7. `import "server-only"` and the data access layer

A data access layer (DAL) puts the database reads in one place, so pages call a function instead of writing queries. `import "server-only"` turns "this code stays on the server" into a rule the build enforces.

**The marker.** It goes at the top of the file. If a Client Component imports this file, directly or through another file, the build fails with a clear error.

```ts
// lib/data/events.ts
import "server-only";

import connectDB from "@/lib/mongodb";
import { Event, type EventFields } from "@/database";

export async function getEventBySlug(slug: string): Promise<EventRecord | null> {
  // ...queries MongoDB
}
```

**Why it's needed when Server Components already run on the server.** Server Components are the default, but a plain `.ts` file has no side of its own. It's bundled into whichever side imports it. Without the marker, nothing states that this file must never reach a Client Component. A mistake might only show up as a confusing Mongoose or `node:fs` bundling error, or not at all if the file only uses code that also runs in a browser.

**The correct way to use it.** A Server Component calls the function and passes the result down as props.

```tsx
// A Server Component (no "use client")
const event = await getEventBySlug(slug);
if (!event) notFound();
return <EventContent event={event} />;
```

**Three similar-looking markers do different jobs.**

| Marker | What it does |
|---|---|
| `import "server-only"` | Build error if client code imports this module |
| `"use server"` | Turns exported async functions into Server Actions, which are public endpoints the browser can call (see section 2) |
| `"use client"` | Marks where client code starts: this module and what it imports go in the browser bundle |

`"use server"` would be the wrong choice here. It would make each database read a public endpoint, while `server-only` keeps these functions internal.

**What it doesn't do.** It's a build-time check on imports. It isn't authentication or authorization, so permission checks are still needed.

**Interview Q:** *What does `import "server-only"` do?* It makes the build fail if the module ends up in client code. It's used on files that touch the database, secrets or the file system.

**Interview Q:** *`server-only` vs `"use server"`?* `server-only` keeps a module away from the client and exposes nothing. `"use server"` does the opposite kind of job: it exposes functions so the client can call them over the network.

**Interview Q:** *Why not fetch your own API route from a Server Component?* The Server Component is already on the server, so calling a Route Handler adds a pointless network round trip and needs an absolute URL. Calling the data function directly is faster and fully typed.

**Interview Q:** *Can a Client Component get data from a `server-only` module?* Not by importing it. A Server Component can load the data and pass it as props, or the client can call a Route Handler.

### 8. Loading page data: failure handling per call, `notFound()` and why reads aren't Server Actions

The event page loads two things, and they don't matter equally. Without the event there's no page. Without similar events, the page is still useful. So each call handles its own failure.

**The required data.** `.catch()` turns a failed read into `null`, and the error is logged on the server. `await fn().catch(...)` does the job of a `try/catch` around one call, and `const` can hold the result directly.

```tsx
// app/events/[slug]/page.tsx
const event = await getEventBySlug(slug).catch((error) => {
  console.error("Loading the event failed:", error);
  return null;
});
```

**Stop if there's nothing to show.** `notFound()` renders the 404 page. After this check, TypeScript knows `event` isn't `null`, so `event.description` is safe below it.

```tsx
if (!event || !event.description || !event.image) return notFound();
```

`notFound()` works by throwing a special error that Next.js catches, and `redirect()` works the same way. If you call either one inside a `try` block, a general `catch` can swallow it and the 404 never happens. Here the `.catch()` wraps only the database call, so `notFound()` stays outside it.

**The optional data.** A failure gives an empty list, so the rest of the page still renders. The `: EventRecord[]` on the callback states the type of the fallback, because a bare `[]` says nothing about what it would hold.

```tsx
const similarEvents = await getSimilarEvents(slug).catch((error): EventRecord[] => {
  console.error("Loading similar events failed:", error);
  return [];
});
```

**Why a read shouldn't be a Server Action.** This used to be a `'use server'` function called from the page while rendering.

```ts
// Before: lib/actions/event.actions.ts
'use server';
export const getSimilarEventsBySlug = async ({ slug }: { slug: string }) => { /* query */ };
```

A Server Action is a public endpoint, and it's designed for changing data. Frameworks may run actions one at a time, and their results can't be cached like a normal read. A page rendering on the server doesn't need a network-callable function at all. It can call a `server-only` data function directly (see section 7), which is also one less endpoint to protect.

| Need | Use |
|---|---|
| Read data while rendering on the server | A data function in the data access layer |
| Change data from a form or button | A Server Action, with validation and permission checks |
| Serve other clients, or fetch from the browser | A Route Handler |

**Interview Q:** *Should a page treat every data failure the same way?* No. Decide per call. If the page can't exist without the data, show a not-found or error state. If the data is optional, use a fallback and log the error so one failed section doesn't take down the page.

**Interview Q:** *Why shouldn't you call `notFound()` or `redirect()` inside a `try/catch`?* They work by throwing. A general `catch` can intercept that error, so the navigation never happens. Call them outside the `try`, or rethrow.

**Interview Q:** *What's the type of `error` in `.catch()` compared to a `catch` block?* In `.catch()` it's `any`. In a `catch` block under strict TypeScript it's `unknown`, which forces you to check it first.

**Interview Q:** *Why not load both with `Promise.all`?* It would be faster for a valid event, because the queries run at the same time. Running them in order avoids a wasted similar-events query when the event doesn't exist. Use `Promise.all` when the calls don't depend on each other and the extra work is acceptable.

**Interview Q:** *When would you use a Server Action, and when a data function?* Use a Server Action for changes triggered by the user. Use a data function for reading data during server rendering.

### 9. Three ways to reach the database: Route Handlers, Server Actions and data functions

StackMeet uses all three, and I kept events on API routes and bookings on a Server Action on purpose, to compare them on real features.

**Route Handler: the browser sends a real HTTP request.** The create form runs in the browser, so it calls a URL and has to check the response itself. `res.json()` has no type information, so TypeScript can't check what comes back.

```ts
// components/createEvent/CreateEventForm.tsx (client component)
const res = await fetch('/api/events', { method: 'POST', body: payload });

const data = await res.json();
if (!res.ok) {
  toast.error(data.message || 'Failed to create event.', { id: toastId });
  return;
}
```

**Server Action: the browser calls a function.** The same kind of job, but there's no URL or `fetch` to write. Next.js creates the network call, and the argument and result types carry through (see section 2).

```ts
// components/events/BookEvent.tsx (client component)
const result = await createBooking({ eventId, slug, email: normalizedEmail });
```

**Data function: the server calls a function.** The page is already on the server, so no network is involved (see section 7).

```tsx
// components/home/NextUp.tsx (Server Component)
const events = await getUpcomingEvents({ q, where, mode, limit: NEXT_UP_LIMIT });
```

| | Route Handler | Server Action | Data function |
|---|---|---|---|
| Has a URL you choose | Yes | No, Next.js generates the endpoint | No |
| Who can call it | Anything that can send HTTP | Your own React code, but the endpoint is public | Server code only |
| Types carry through the call | No | Yes | Yes |
| Reachable from the internet | Yes | Yes | No |
| Needs validation and permission checks | Yes | Yes | Permission checks if the data is private |
| Best for | Outside callers, webhooks, file downloads, sign-in handling | Changes from your own forms and buttons | Reading data while rendering |

**The mistake I removed: a server fetching itself.** The home page used to call its own API route, then filter the results in JavaScript.

```ts
// Before: app/page.tsx (a Server Component)
const BASE_URL = process.env.NEXT_PUBLIC_BASE_URL;
const allEvents = (await (await fetch(`${BASE_URL}/api/events`)).json()).events ?? [];
```

The request left the server only to come back to the same server. It cost an extra network round trip, needed the site's address to be configured correctly, lost its types, and downloaded every event just to show nine. Switching to data functions removed the need for `NEXT_PUBLIC_BASE_URL` completely. That variable also had to be set before building, because `NEXT_PUBLIC_` values are fixed at build time ([Talent Pipeline notes #46](../frontend/typescript-react-nextjs-from-Talent-Pipeline-Tracker.md#46-environment-variables-next_public_)).

**Why the browser can use `/api/events` but the server can't.** The browser resolves a relative path against the page it's on. A Server Component isn't on a page in that sense, so `fetch` on the server needs a full URL.

**Interview Q:** *Server Action vs API route?* A Server Action suits changes from your own React forms and buttons: no URL to manage, and the types carry through. An API route suits anything that needs a real URL, such as a mobile app, a webhook or another service. Both are public endpoints, so both need validation and permission checks.

**Interview Q:** *Do Server Actions replace API routes?* No. A Server Action is tied to your React app, while an API route is a general HTTP endpoint that any client can call.

**Interview Q:** *Why not fetch your own API from a Server Component?* It adds a network round trip to the same server, needs an absolute URL and loses type safety. Put the query in a shared function, then have both the page and the route call it.

**Interview Q:** *Why does `fetch('/api/events')` work in the browser but not in a Server Component?* The browser can resolve a relative path against the current page's address. On the server there's no current page to resolve it against, so it needs an absolute URL.
