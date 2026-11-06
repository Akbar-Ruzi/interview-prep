# Interview Notes: Next.js Full Stack, Auth, MongoDB and Mongoose

> Every snippet is from my project (StackMeet, a developer events app: Next.js 16, React 19, TypeScript, Mongoose 8, Auth.js v5), shortened where needed.
> Snippets marked **Planned fix** show the fix I would make. They aren't in the project yet.
> For each concept: what it is, the project code, and a likely interview question.

## Contents

- [MongoDB and Mongoose](#mongodb-and-mongoose)
  1. Schemas, validation and TypeScript types
  2. `models.X || model()`: avoiding `OverwriteModelError`
  3. Caching the database connection
  4. Pre-save hooks: generating a slug
  5. `findOneAndUpdate` skips save hooks
  6. Indexes: unique, compound and redundant
  7. A unique index beats "check, then insert"
  8. Atomic updates: enforcing a limit under parallel requests
  9. `.lean()`
- [Authentication and authorization](#authentication-and-authorization)
  10. Hashing passwords with bcrypt
  11. Credentials sign-in with `authorize`
  12. JWT sessions: passing `id` and `role` through callbacks
  13. Module augmentation: typing the session
  14. Authentication vs authorization (401 vs 403)
  15. Mass assignment: never trust the request body
- [Next.js full stack](#nextjs-full-stack)
  16. Route Handlers and HTTP status codes
  17. Server Actions and returning a result object
  18. The `satisfies` operator
  19. Uploading a file: wrapping a callback API in a Promise
  20. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"
  21. Using `rewrites` as a reverse proxy for analytics
- [Project decisions and tooling](#project-decisions-and-tooling)
  22. Protecting a free database: why one limit isn't enough
  23. Guest bookings: an unverified email doesn't prove ownership
  24. `.env.example` and un-ignoring a file in `.gitignore`
  25. shadcn/ui: copied code, not a dependency
  26. Git: uncommitted changes and branch prefixes

---

## MongoDB and Mongoose

### 1. Schemas, validation and TypeScript types

MongoDB doesn't enforce a structure by default, so a Mongoose schema adds one at the application level. The TypeScript interface checks types at compile time, while the schema validates data at runtime. You need both.

```ts
// database/event.model.ts
export interface EventFields {
  title: string;
  mode: 'online' | 'offline' | 'hybrid';
  agenda: string[];
  // ...
}

// The stored document: the fields plus what MongoDB and Mongoose add
interface IEvent extends EventFields, Document {
  createdAt: Date;
  updatedAt: Date;
  createdBy: Types.ObjectId; // reference to the User who created the event
}

const EventSchema = new Schema<IEvent>(
  {
    title: {
      type: String,
      required: [true, 'Title is required'],
      trim: true,
      maxlength: [100, 'Title cannot exceed 100 characters'],
    },
    mode: {
      type: String,
      required: [true, 'Mode is required'],
      enum: {
        values: ['online', 'offline', 'hybrid'],
        message: 'Mode must be either online, offline, or hybrid',
      },
    },
    agenda: {
      type: [String],
      required: [true, 'Agenda is required'],
      validate: {
        validator: (v: string[]) => v.length > 0,
        message: 'At least one agenda item is required',
      },
    },
    createdBy: { type: Schema.Types.ObjectId, ref: 'User', required: true, index: true },
  },
  { timestamps: true } // adds createdAt and updatedAt automatically
);
```

- `required` on an array isn't enough, because an empty array still passes. That's why `agenda` has a custom validator.
- `ref: 'User'` enables `.populate()`. It doesn't create a foreign key, so MongoDB won't stop a reference to a deleted user.
- Splitting `EventFields` from `IEvent` gives a plain data type for props, separate from the full database document.

**Interview Q:** *TypeScript already has types, so why do you need a schema?* TypeScript types are removed at compile time, so they can't check data that arrives at runtime, such as a request body. The schema does that check before anything is saved.

### 2. `models.X || model()`: avoiding `OverwriteModelError`

```ts
// database/event.model.ts
const Event = models.Event || model<IEvent>('Event', EventSchema);
export default Event;
```

Mongoose keeps a registry of compiled models and throws `OverwriteModelError` if the same name is compiled twice. Next.js hot reload re-runs modules in development, so this line reuses the existing model when there is one.

**Interview Q:** *Why not just call `model('Event', schema)`?* Because any module that runs again, such as after a hot reload, would try to register the model twice and throw.

### 3. Caching the database connection

Opening a MongoDB connection is slow and connections are limited. Without a cache, every hot reload or request could open a new one until the limit is reached.

```ts
// lib/mongodb.ts
const cached: MongooseCache = global.mongoose || { conn: null, promise: null };

if (!global.mongoose) {
  global.mongoose = cached;
}

async function connectDB(): Promise<typeof mongoose> {
  if (cached.conn) return cached.conn;

  if (!cached.promise) {
    if (!MONGODB_URI) throw new Error('Please define the MONGODB_URI environment variable');

    cached.promise = mongoose.connect(MONGODB_URI, { bufferCommands: false });
  }

  try {
    cached.conn = await cached.promise;
  } catch (error) {
    cached.promise = null; // allow a retry after a failure
    throw error;
  }

  return cached.conn;
}
```

- **Why `global`:** it survives hot reloads, which reset module-level variables. A serverless instance can also reuse it across requests while it stays warm.
- **Why cache the promise as well:** if two requests arrive during connection, both await the same promise instead of opening two connections.
- **Why reset the promise on failure:** otherwise every later request would keep awaiting the same rejected promise, and the app could never reconnect.
- **`bufferCommands: false`:** queries fail right away if there's no connection, rather than waiting in a queue.

**Interview Q:** *How do you manage database connections in serverless?* Cache one connection per instance on `global`, and cache the in-progress promise so concurrent requests share it. Each instance still holds its own connection, so many instances can still use up the limit.

### 4. Pre-save hooks: generating a slug

Middleware (hooks) runs logic automatically around an operation. This hook keeps derived data correct wherever the event is saved.

```ts
// database/event.model.ts
EventSchema.pre('save', function (next) {
  if (this.isModified('title') || this.isNew) {
    this.slug = generateSlug(this.title);
  }
  if (this.isModified('date')) {
    this.date = normalizeDate(this.date);
  }
  next();
});

function generateSlug(title: string): string {
  return title
    .toLowerCase()
    .trim()
    .replace(/[^a-z0-9\s-]/g, '') // remove special characters
    .replace(/\s+/g, '-')         // spaces to hyphens
    .replace(/-+/g, '-')          // collapse repeated hyphens
    .replace(/^-|-$/g, '');       // trim leading and trailing hyphens
}
```

- It uses a regular `function`, not an arrow function, because the hook needs `this` to be the document.
- `isModified` avoids unnecessary work when the field didn't change.
- **Design trade-off:** regenerating the slug when a title changes breaks links people have already shared. I decided to keep the original slug on edit.

**Interview Q:** *Why can't a Mongoose hook be an arrow function?* Arrow functions don't have their own `this`, and document middleware uses `this` to access the document.

### 5. `findOneAndUpdate` skips save hooks

This was a real bug in my project.

```ts
// app/api/events/[slug]/route.ts: the bug
const updatedEvent = await Event.findOneAndUpdate(
  { slug: result.slug },
  { $set: body },
  { new: true, runValidators: true }
);
```

`pre('save')` is **document** middleware, while `findOneAndUpdate` is **query** middleware. It updates MongoDB directly and never calls `save()`, so the date normalization in the hook doesn't run.

**Planned fix:**

```ts
const event = await Event.findOne({ slug });
event.set(allowedFields); // only the fields users may edit
await event.save();       // runs validators and pre('save')
```

- `save()` loads the document first and takes two database calls. It's a good default when hooks hold business logic.
- `findOneAndUpdate` is one atomic operation, which suits counters and other concurrent changes.
- Update validators are off by default, so `runValidators: true` is needed for them to run at all.

**Interview Q:** *When would you use `save()` over `findOneAndUpdate`?* Use `save()` when you need document hooks and full validation. Use `findOneAndUpdate` when the update has to be atomic and doesn't depend on hooks.

### 6. Indexes: unique, compound and redundant

An index lets MongoDB find documents without scanning the whole collection. The cost is extra storage and slightly slower writes.

```ts
// database/booking.model.ts
BookingSchema.index({ eventId: 1 });
BookingSchema.index({ eventId: 1, createdAt: -1 }); // an event's bookings, newest first
BookingSchema.index({ email: 1 });                  // a user's bookings
BookingSchema.index({ eventId: 1, email: 1 }, { unique: true, name: 'uniq_event_email' });
```

- **Prefix rule:** a compound index can serve queries on its leading fields. `{ eventId: 1, createdAt: -1 }` already covers queries on `eventId`, so the first index above is redundant.
- **Field order matters:** a common guideline is Equality, then Sort, then Range (ESR).
- **Duplicate declarations:** `unique: true` on a field already creates an index. Adding `schema.index()` for the same field triggered this warning in my build:

```text
[MONGOOSE] Warning: Duplicate schema index on {"email":1} found.
```

**Interview Q:** *Does an index on `(a, b)` help a query that only filters on `b`?* Generally no. It helps queries on `a`, or on `a` and `b`, because the index is sorted by `a` first.

### 7. A unique index beats "check, then insert"

Checking first and then inserting has a race condition, because two requests can both pass the check. Only the database can guarantee uniqueness.

```ts
// Booking schema: one booking per email for each event
BookingSchema.index({ eventId: 1, email: 1 }, { unique: true, name: 'uniq_event_email' });

// lib/actions/booking.action.ts
try {
  await Booking.create({ eventId, email: normalizedEmail });
  return { success: true };
} catch (e) {
  if (typeof e === 'object' && e !== null && 'code' in e && (e as { code?: number }).code === 11000) {
    return { success: false, message: 'This email is already registered for this event.' };
  }
  return { success: false, message: 'Unable to create booking right now. Please try again.' };
}
```

My sign-up route shows the problem:

```ts
// app/api/auth/signup/route.ts
const existingUser = await User.findOne({ email }).lean();
if (existingUser) {
  return NextResponse.json({ message: 'Email already exists' }, { status: 409 });
}
// A second request can reach this line at the same time
const createdUser = await User.create({ name, email, password: hashedPassword, role: 'user' });
```

The unique index still blocks the duplicate, so the data stays correct. But the second request gets a 500 instead of a 409, so it should also catch error `11000`.

**Interview Q:** *What is error code 11000?* MongoDB's duplicate key error, returned when a write violates a unique index. Catching it and returning a clear message is the standard way to handle duplicates.

### 8. Atomic updates: enforcing a limit under parallel requests

My limit on events per user has the same race, and a unique index can't express "at most N".

```ts
// app/api/events/route.ts: parallel requests can all pass this check
const existingCount = await Event.countDocuments({ createdBy: userId });
if (existingCount >= 2) {
  return NextResponse.json({ message: 'You have reached the maximum limit of 2 events.' }, { status: 403 });
}
// ...later: await Event.create(eventPayload);
```

**Planned fix:** put the check and the increment in one operation.

```ts
const user = await User.findOneAndUpdate(
  { _id: userId, eventCount: { $lt: MAX_EVENTS } }, // only matches if under the limit
  { $inc: { eventCount: 1 } },
  { new: true }
);
if (!user) return limitReachedResponse();

try {
  await Event.create(eventPayload);
} catch (error) {
  await User.updateOne({ _id: userId }, { $inc: { eventCount: -1 } }); // give the slot back
  throw error;
}
```

MongoDB writes to a single document are atomic, so parallel requests can't both pass the `$lt` check for the last slot. The counter and the event are separate writes, so a failed create has to give the slot back. A transaction is the stricter alternative.

**Interview Q:** *How do you enforce a limit safely under concurrency?* Combine the check and the update into one atomic operation, such as a conditional `$inc`, rather than reading first and writing later.

### 9. `.lean()`

```ts
const event = await Event.findOne({ slug: sanitizedSlug }).lean();
```

`.lean()` returns plain JavaScript objects instead of full Mongoose documents, so queries use less memory and CPU. Lean results don't have `save()`, virtuals or getters, so use it for read-only data, not for documents you'll change.

**Interview Q:** *When shouldn't you use `.lean()`?* When you need to change and `save()` the document, or when you rely on virtuals, getters or document methods.

---

## Authentication and authorization

### 10. Hashing passwords with bcrypt

```ts
// app/api/auth/signup/route.ts
const hashedPassword = await bcrypt.hash(password, 12);

// auth.ts
const isPasswordValid = await bcrypt.compare(password, user.password);
```

- **Hashing, not encryption:** hashing is one-way, so a stolen database doesn't directly reveal passwords.
- **Salt:** bcrypt adds a random salt to each hash, so two users with the same password get different hashes. The salt is stored inside the hash, which is why `compare` doesn't need it passed in.
- **Cost factor (12):** the deliberately slow work makes brute-force attacks expensive. Raising it by 1 doubles the work.
- **Why not SHA-256:** it's designed to be fast, which helps attackers guess passwords quickly.
- **Bug I found:** bcrypt only uses the first 72 **bytes**, but my check counted characters, and non-ASCII characters take more than one byte. The correct check is `Buffer.byteLength(password, 'utf8')`.

**Interview Q:** *How should passwords be stored?* Never as plain text. Store a salted hash from a slow algorithm designed for passwords, such as bcrypt or Argon2, and verify logins with its compare function.

### 11. Credentials sign-in with `authorize`

```ts
// auth.ts
Credentials({
  async authorize(credentials) {
    const email = credentials?.email;
    const password = credentials?.password;

    if (typeof email !== 'string' || typeof password !== 'string') return null;

    await connectDB();

    const user = await User.findOne({ email: email.toLowerCase().trim() });
    if (!user) return null;

    const isPasswordValid = await bcrypt.compare(password, user.password);
    if (!isPasswordValid) return null;

    // Never include the password hash in what's returned
    return { id: user._id.toString(), name: user.name, email: user.email, role: user.role };
  },
}),
```

- Returning `null` for both "no such user" and "wrong password" gives the attacker the same failure, so sign-in doesn't reveal which emails exist.
- `typeof` checks are needed because the values come from the client and aren't guaranteed to be strings. This also blocks objects used for NoSQL injection, such as `{ "$ne": null }`.
- **Remaining leaks:** my sign-up route still reveals existing emails through its 409 response. A failed lookup also returns faster than a bcrypt compare, which can expose the same thing through timing.

**Interview Q:** *Why not tell users which part of the login was wrong?* It would let attackers discover which emails are registered, and then target those accounts.

### 12. JWT sessions: passing `id` and `role` through callbacks

```ts
// auth.ts
session: { strategy: 'jwt' },
callbacks: {
  async jwt({ token, user }) {
    if (user) { // only present at sign-in
      token.id = user.id as string;
      token.role = (user as { role?: 'user' | 'admin' }).role ?? 'user';
    }
    return token;
  },
  async session({ session, token }) {
    if (session.user) {
      session.user.id = token.id as string;
      session.user.role = (token.role as 'user' | 'admin') ?? 'user';
    }
    return session;
  },
},
```

The flow is: `authorize` returns the user, `jwt` copies data into the token, and `session` exposes it to the app. The token is stored in a cookie, so there's no session lookup in the database on each request.

| | JWT session | Database session |
|---|---|---|
| Storage | Cookie on the client | Row or document on the server |
| Database call per request | No | Yes |
| Revoking access immediately | Hard | Easy, delete the session |
| Changing a role | Applies when the token is next refreshed | Applies right away |

**Interview Q:** *What's the main downside of JWT sessions?* You can't easily revoke one before it expires, and data inside it, like a role, can go stale. Sensitive actions may need a fresh database check.

### 13. Module augmentation: typing the session

Auth.js's default session type doesn't include `id` or `role`. Module augmentation adds them to the library's types without changing the library.

```ts
// types/next-auth.d.ts
import { DefaultSession } from 'next-auth';

declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      role: 'user' | 'admin';
    } & DefaultSession['user']; // keep name, email and image
  }
}
```

**Interview Q:** *How do you add a property to a third-party library's types?* Use `declare module` to extend its interface. Interfaces with the same name merge, which is one reason libraries expose interfaces for this.

### 14. Authentication vs authorization (401 vs 403)

- **Authentication:** who are you? Failing returns **401**.
- **Authorization:** are you allowed to do this? Failing returns **403**.

```ts
// app/api/events/[slug]/route.ts
async function getAuthorizedEvent(params: Promise<{ slug: string }>): Promise<AuthorizedEventResult> {
  const session = await auth();
  if (!session?.user) {
    return { error: NextResponse.json({ message: 'Unauthorized' }, { status: 401 }) };
  }

  await connectDB();

  const { slug } = await params;
  const event = await Event.findOne({ slug: slug.trim().toLowerCase() });
  if (!event) {
    return { error: NextResponse.json({ message: 'Event not found' }, { status: 404 }) };
  }

  const isAdmin = session.user.role === 'admin';
  const isOwner = event.createdBy.toString() === session.user.id;

  if (!isAdmin && !isOwner) {
    return { error: NextResponse.json({ message: 'You are not allowed to modify this event.' }, { status: 403 }) };
  }

  return { event, slug };
}
```

- Hiding a button isn't security, because anyone can call the API directly. The check has to be on the server.
- `.toString()` is needed because `createdBy` is an `ObjectId`, and comparing it with a string using `===` is always false.
- **Bug I found:** older events had no `createdBy`, so `.toString()` threw and returned a 500 even for admins. `event.createdBy?.toString()` handles that case.

**Interview Q:** *What's the difference between 401 and 403?* 401 means the user isn't authenticated, so signing in might help. 403 means they're known but don't have permission.

### 15. Mass assignment: never trust the request body

Passing a request body straight into a database update lets the client change fields it should never touch.

```ts
// The bug: PATCH /api/events/[slug]
const body = await request.json();
await Event.findOneAndUpdate({ slug }, { $set: body }, { new: true, runValidators: true });
```

An owner could send `{ "createdBy": "<someone-else's id>" }` and hand the event to another user, which also gets around the per-user event limit.

**Planned fix:** allow only a fixed list of fields.

```ts
const EDITABLE_FIELDS = ['title', 'description', 'overview', 'venue', 'date', 'time'] as const;

const updates = Object.fromEntries(
  EDITABLE_FIELDS.filter((key) => key in body).map((key) => [key, body[key]])
);
```

Create does this correctly, because the owner comes from the session rather than the form:

```ts
// app/api/events/route.ts
const eventPayload = { ...eventFields, image: uploadResult.secure_url, createdBy: userId };
```

**Interview Q:** *What is a mass assignment vulnerability?* The server copies client input directly onto a model, so an attacker can set fields they shouldn't, such as `role` or an owner id. Prevent it with an allowlist of fields.

---

## Next.js full stack

### 16. Route Handlers and HTTP status codes

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

### 17. Server Actions and returning a result object

A Server Action is a server function that a client component can call directly, without you writing a fetch or an API route.

```ts
// lib/actions/booking.action.ts
'use server';

export const createBooking = async ({ eventId, email }: CreateBookingParams) => {
  await connectDB();
  // ...
  return { success: true } satisfies CreateBookingResult;
};

// components/BookEvent.tsx (client component)
const result = await createBooking({ eventId, slug, email: normalizedEmail });
if (result.success) {
  setSubmitted(true);
} else {
  setErrorMessage(result.message ?? 'Booking creation failed. Please try again.');
}
```

- **Result object instead of `throw`:** expected failures, like a duplicate booking, are normal outcomes the UI can display. In production, Next.js also hides the message of errors thrown from Server Actions.
- **Security:** a Server Action is a public POST endpoint, so it needs the same validation and permission checks as an API route. Client-side validation alone isn't enough.

**Interview Q:** *Server Action or Route Handler?* Server Actions suit mutations from your own UI. Route Handlers suit endpoints that outside clients, webhooks or other apps need to call.

### 18. The `satisfies` operator

```ts
return { success: false, message: 'Email is required.' } satisfies CreateBookingResult;
```

`satisfies` checks a value against a type without changing the value's inferred type. A wrong or misspelled field is a compile error.

| | Checks the shape | Keeps the specific inferred type |
|---|---|---|
| `const x: T = value` | Yes | No, `x` becomes `T` |
| `value as T` | Only loosely, and can hide mistakes | No, treated as `T` |
| `value satisfies T` | Yes | Yes |

**Interview Q:** *Why use `satisfies` instead of `as`?* `as` tells the compiler to trust you and can hide mistakes. `satisfies` makes the compiler verify the value.

### 19. Uploading a file: wrapping a callback API in a Promise

Cloudinary's upload stream reports its result through a callback, so wrapping it in a Promise allows `await`.

```ts
// app/api/events/route.ts
const formData = await req.formData();
const imageValue = formData.get('image');
if (!(imageValue instanceof File)) {
  return NextResponse.json({ message: 'Image file is required' }, { status: 400 });
}

const buffer = Buffer.from(await imageValue.arrayBuffer());

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

- Files use `FormData` because JSON can't carry binary data without encoding it.
- The database stores only the image's URL, not the file itself.
- The whole file is held in memory as a `Buffer`, so the type and size should be checked before this point. My code doesn't do that yet, and it's in my plan.

**Interview Q:** *How do you turn a callback API into a Promise?* Wrap it in `new Promise`, call `reject` on an error and `resolve` on success. `util.promisify` does this for standard Node-style callbacks.

### 20. `cacheComponents`: "Uncached data was accessed outside of `<Suspense>`"

This was a real production build failure, and it only appeared in `next build`.

```text
Error: Route "/events/[slug]": Uncached data was accessed outside of <Suspense>.
```

With `cacheComponents: true`, Next.js prerenders a static shell of each page. Anything only known at request time, such as a `fetch` or `await params`, must be cached with `"use cache"` or sit inside `<Suspense>`, so the shell can be sent first.

```tsx
// app/events/[slug]/page.tsx: before (the whole page awaits request-time data)
const EventDetailsPage = async ({ params }: EventPageProps) => {
  const { slug } = await params;
  const res = await fetch(`${BASE_URL}/api/events/${slug}`);
  // ...
};

// after: the page is synchronous, and the data work moves inside the boundary
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

### 21. Using `rewrites` as a reverse proxy for analytics

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

```ts
// instrumentation-client.ts
posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
  api_host: '/ingest', // same-origin path, not posthog.com
});
```

Requests go to my own domain and Next.js forwards them. Ad blockers often block known analytics domains, so a same-origin path makes the tracking more reliable.

- A **redirect** tells the browser to go to a new URL, and the address changes.
- A **rewrite** happens on the server, and the browser's URL stays the same.

**Interview Q:** *What's the difference between a redirect and a rewrite?* A redirect sends the browser to a new URL with a 3xx response. A rewrite serves the content from another location while keeping the original URL.

---

## Project decisions and tooling

These came from questions I asked while building StackMeet.

### 22. Protecting a free database: why one limit isn't enough

My first idea was to limit events per user, so a public site couldn't push my free MongoDB plan over its storage limit. That alone doesn't work, because sign-up has no email verification and one person can create many accounts. The real protection is a set of limits that covers each way data can grow.

| Limit (planned) | Value | What it protects against |
|---|---|---|
| Events per user | 3 | One account filling the site |
| Total events | 100 | Many fake accounts, because it holds however many accounts exist |
| Bookings per event | 200 | Guests booking with made-up emails |
| Upcoming events per email | 3 | One email holding many spots |
| Cover image size | 2 MB | Uploads using up image storage |

Together these cap the site at 100 x 200 = 20,000 bookings, so storage has a known upper limit.

**Only upcoming events count toward the per-email limit.** If past events counted, someone who attended 3 events could never book again.

**Limits don't stop everything.** Someone could still fill an event's 200 spots with fake emails, and a per-email limit won't help because they can keep changing the address. That needs rate limiting or email verification, which are planned for after the prototype.

**Interview Q:** *How would you stop users from running up your costs on a public app?* Find every way data can grow and cap each one, including a global cap that doesn't depend on user identity, because accounts are cheap to create. Then add rate limiting and email verification, and set up billing alerts or a free tier that can't charge you.

### 23. Guest bookings: an unverified email doesn't prove ownership

Guests can book with only a name and email, so `userId` is optional.

```ts
// lib/actions/booking.action.ts
const session = await auth();

await Booking.create({
  eventId,
  userId: session?.user?.id, // Guests have no session, so this stays empty
  name: cleanName,
  email: normalizedEmail,
  phone: cleanPhone || undefined,
});
```

I asked how a guest could see their booking later. It seems natural to show it once they sign up with the same email, but that's a security hole: emails aren't verified, so anyone could register with someone else's address and see or cancel that person's bookings. In the prototype, "My events" shows only bookings made while signed in.

| Person | See the booking? | Cancel it? |
|---|---|---|
| Signed-in user | Yes, on My events | Yes |
| Guest, prototype | Only the success message | No |
| Guest, once email is added (planned) | Yes, through an emailed link | Yes |

**The trade-off:** a guest can't cancel, so their booking counts toward the email's limit until the event ends. I accepted that to ship the first prototype sooner.

**Interview Q:** *A user signs up with an email that already has guest orders. Should you link them automatically?* Only after verifying the email. Otherwise, signing up with someone else's address gives access to their data.

### 24. `.env.example` and un-ignoring a file in `.gitignore`

`.env` holds real secrets and must never be committed. `.env.example` is committed with placeholder values, so anyone who clones the repo can see which settings the app needs.

```gitignore
# .gitignore
.env*
!.env.example
```

`.env*` also matches `.env.example`, so the file would stay off GitHub without the second line. A `!` pattern re-includes a file, and when several patterns match, the last one wins, so the order matters.

```bash
# .env.example
MONGODB_URI=your_connection_string
AUTH_SECRET=your_secret
NEXT_PUBLIC_POSTHOG_KEY=your_key
```

**Interview Q:** *What does the `NEXT_PUBLIC_` prefix do?* It puts the value into the browser bundle, so anyone can read it. Use it only for values that are safe to expose, and keep database URLs and auth secrets unprefixed so they stay on the server.

**Interview Q:** *You committed a secret by accident. Is deleting it in a new commit enough?* No, it's still in the Git history. Rotate the secret first, then clean the history if needed.

### 25. shadcn/ui: copied code, not a dependency

I asked whether I needed shadcn/ui, and I didn't. It isn't a normal component library: its command copies each component's source into your project, so you own the code and can restyle it.

```bash
npx shadcn@latest add alert-dialog sheet
```

StackMeet doesn't use it. The design is already distinctive and most of the UI is simple, so the site uses Tailwind. The delete and cancel popups use the native `<dialog>` element instead, which handles much of the keyboard and focus behavior that's easy to get wrong.

**Interview Q:** *When would you use a UI library rather than build it yourself?* For components with hard accessibility behavior, such as dialogs and menus. Simple things like cards and buttons rarely need one, and every extra library is more code to maintain.

### 26. Git: uncommitted changes and branch prefixes

Creating a branch doesn't require a commit first. Uncommitted changes aren't tied to a branch, so they stay in your working folder and come with you.

```bash
git switch -c feat/site-redesign   # uncommitted changes come along
git status                         # they still show as changed
```

Committing on the new branch leaves `main` unchanged until you merge. Switching to an existing branch is different: Git refuses if that switch would overwrite your uncommitted changes, so you commit or stash them first.

Prefixes like `feat/` are a naming convention that Git doesn't require. They show what kind of work a branch contains.

| Prefix | Used for |
|---|---|
| `feat/` | New features or redesigns |
| `fix/` | Bug fixes |
| `chore/` | Maintenance, like renames or config |
| `docs/` | Documentation |

**Interview Q:** *What does `git stash` do?* It saves your uncommitted changes and cleans the working folder, so you can switch branches and bring the changes back later with `git stash pop`.
