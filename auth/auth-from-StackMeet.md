# Interview Notes: Authentication and Authorization

> Every snippet is from my project (StackMeet, a developer events app: Next.js 16, React 19, TypeScript, Mongoose 8, Auth.js v5), shortened where needed.
> Snippets marked **Planned fix** show the fix I would make. They aren't in the project yet.
> For each concept: what it is, the project code, and a likely interview question.
> Other notes from this project: [Next.js API and server code](../backend/nextjs-api-from-StackMeet.md), [Next.js rendering and config](../frontend/nextjs-from-StackMeet.md) and [MongoDB and Mongoose](../databases/mongodb-mongoose-from-StackMeet.md).

## Contents

- [Authentication and authorization](#authentication-and-authorization)
  1. Hashing passwords with bcrypt
  2. Credentials sign-in with `authorize`
  3. JWT sessions: passing `id` and `role` through callbacks
  4. Module augmentation: typing the session
  5. Authentication vs authorization (401 vs 403)
  6. Mass assignment: never trust the request body
  7. The sign-in form: `useAuthForm`, `signIn` and what happens after
- [Project decisions](#project-decisions)
  8. Guest bookings: an unverified email doesn't prove ownership

---

## Authentication and authorization

### 1. Hashing passwords with bcrypt

**Hash the password at sign-up.**

- **Hashing, not encryption:** hashing is one-way, so a stolen database doesn't directly reveal passwords.
- **Salt:** bcrypt adds a random salt to each hash, so two users with the same password get different hashes.
- **Cost factor (12):** the deliberately slow work makes brute-force attacks expensive. Raising it by 1 doubles the work.
- **Why not SHA-256:** it's designed to be fast, which helps attackers guess passwords quickly.
- **Bug I found and fixed:** bcrypt only uses the first 72 **bytes**, but my check counted characters, and non-ASCII characters take more than one byte. For example, 30 emoji are only 60 in `password.length` but take 120 bytes, so bcrypt would have ignored part of the password. The route now uses `Buffer.byteLength(password, 'utf8')`, and also limits passwords to 64 characters.

```ts
// app/api/auth/signup/route.ts
const hashedPassword = await bcrypt.hash(password, 12);
```

**Check the password at sign-in.** The salt is stored inside the hash, which is why `compare` doesn't need it passed in.

```ts
// auth.ts
const isPasswordValid = await bcrypt.compare(password, user.password);
```

**Interview Q:** *How should passwords be stored?*

Never as plain text. Store a salted hash from a slow algorithm designed for passwords, such as bcrypt or Argon2, and verify logins with its compare function.

### 2. Credentials sign-in with `authorize`

**The main idea.** `auth.ts` is the one place that configures authentication. `NextAuth()` takes the settings and returns four tools, which the rest of the app imports. Destructuring pulls all four out in one statement.

```ts
// auth.ts
export const { handlers, auth, signIn, signOut } = NextAuth({
  session: { strategy: 'jwt' },
  pages: { signIn: '/auth/sign-in' }, // use my page, not the built-in one
  providers: [Credentials({ /* ... */ })],
  callbacks: { /* ... */ },
});
```

| Export | What it's for |
|---|---|
| `handlers` | The `GET` and `POST` functions that serve Auth.js's own endpoints |
| `auth` | Reads the current session on the server |
| `signIn`, `signOut` | Start or end a session from server code |

**Connecting the endpoints.** `[...nextauth]` is a catch-all route, so every URL under `/api/auth/` reaches Auth.js, such as `/api/auth/session` and `/api/auth/callback/credentials`. My `/api/auth/signup` folder still works, because a specific route takes priority over a catch-all.

```ts
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth';
export const { GET, POST } = handlers;
```

**Three things called "auth", three different jobs.** The core concept is that in the App Router, a file's location decides whether it gets a URL. The three share a name because they all relate to signing in, but one is the rules, one is what people see, and one is what the browser calls.

| Path | What it is | What it does | Has a URL? |
|---|---|---|---|
| `auth.ts` | Configuration | Defines how passwords are checked and what goes in the session | No |
| `app/auth/` | Pages | `page.tsx` files render `/auth/sign-in` and `/auth/sign-up` | Yes, HTML pages |
| `app/api/auth/` | API endpoints | `route.ts` files handle `/api/auth/signup` and the `[...nextauth]` routes | Yes, HTTP endpoints |

**Why they're different.** Inside `app/`, a folder only becomes a URL when it contains a special file: `page.tsx` for a page, or `route.ts` for an endpoint. `auth.ts` sits outside `app/` because it's plain server code that should never have a URL. It holds the rules, and both of the others import it, so the rules are written once. The project root is the usual Auth.js location but not a requirement, and it's why the import path is `@/auth`.

```text
app/auth/sign-in/page.tsx              the page, calls auth() to check the session
      ↓ imports
auth.ts                                the rules
      ↑ imports
app/api/auth/[...nextauth]/route.ts    the endpoints the browser calls to sign in
```

**What happens during a sign-in.** Each part does its job in turn.

```text
1. /auth/sign-in shows the form                     app/auth/
2. The browser posts to /api/auth/callback/credentials
3. [...nextauth] passes it to the handlers           app/api/auth/
4. authorize() checks the password, cookie is set    auth.ts
5. Later pages call auth(), which reads the cookie   auth.ts
```

**There are two `signIn` functions.** The one exported from `auth.ts` is for server code. Browser components use the one from `next-auth/react`, which calls those endpoints. Importing `auth.ts` into a client component would pull Mongoose and bcrypt toward the browser.

```ts
// hooks/useAuthForm.ts (client code)
import { signIn } from 'next-auth/react';
```

**The outline.** `authorize` receives what the sign-in form sent. The steps below all run inside it.

```ts
// auth.ts
Credentials({
  async authorize(credentials) {
    // Steps 1 to 4
  },
}),
```

**Step 1: Check the input types.** `typeof` checks are needed because the values come from the client and aren't guaranteed to be strings. This also blocks objects used for NoSQL injection, such as `{ "$ne": null }`.

```ts
const email = credentials?.email;
const password = credentials?.password;

if (typeof email !== 'string' || typeof password !== 'string') return null;
```

**Step 2: Find the user.** Returning `null` for both "no such user" and "wrong password" gives the attacker the same failure, so sign-in doesn't reveal which emails exist.

**Remaining leaks:** my sign-up route still reveals existing emails through its 409 response. A failed lookup also returns faster than a bcrypt compare, which can expose the same thing through timing.

```ts
await connectDB();

const user = await User.findOne({ email: email.toLowerCase().trim() });
if (!user) return null;
```

**Step 3: Check the password.** A wrong password returns the same `null`.

```ts
const isPasswordValid = await bcrypt.compare(password, user.password);
if (!isPasswordValid) return null;
```

**Step 4: Return the user.** Never include the password hash in what's returned.

```ts
return { id: user._id.toString(), name: user.name, email: user.email, role: user.role };
```

**Interview Q:** *Why not tell users which part of the login was wrong?*

It would let attackers discover which emails are registered, and then target those accounts.

**Interview Q:** *How does the App Router decide what becomes a URL?*

Folders inside `app/` form the path, but a route only exists when the folder has a `page.tsx` or a `route.ts`. Other files in those folders, and anything outside `app/`, get no URL.

**Interview Q:** *What's the difference between `page.tsx` and `route.ts`?*

`page.tsx` renders UI for a visitor. `route.ts` handles HTTP methods and returns a response, such as JSON. One folder can't have both, because they'd claim the same URL.

**Interview Q:** *Why is the Auth.js config outside `app/`?*

It's shared server code, not a route. Pages and API routes both import it, so the authentication rules live in one place.

**Interview Q:** *What does `[...nextauth]` mean?*

It's a catch-all segment that matches any number of path parts after `/api/auth/`, so one file can serve all of Auth.js's endpoints. A more specific route, like `/api/auth/signup`, takes priority.

### 3. JWT sessions: passing `id` and `role` through callbacks

The flow is: `authorize` returns the user, `jwt` copies data into the token, and `session` exposes it to the app.

**Choose JWT sessions.** The token is stored in a cookie, so there's no session lookup in the database on each request.

```ts
// auth.ts
session: { strategy: 'jwt' },
```

**Copy the data into the token.** `user` is only present at sign-in, so that's when `id` and `role` are copied.

```ts
callbacks: {
  async jwt({ token, user }) {
    if (user) { // only present at sign-in
      token.id = user.id as string;
      token.role = (user as { role?: 'user' | 'admin' }).role ?? 'user';
    }
    return token;
  },
```

**Expose the data to the app.** This copies `id` and `role` from the token onto the session.

```ts
  async session({ session, token }) {
    if (session.user) {
      session.user.id = token.id as string;
      session.user.role = (token.role as 'user' | 'admin') ?? 'user';
    }
    return session;
  },
},
```

| | JWT session | Database session |
|---|---|---|
| Storage | Cookie on the client | Row or document on the server |
| Database call per request | No | Yes |
| Revoking access immediately | Hard | Easy, delete the session |
| Changing a role | Applies when the token is next refreshed | Applies right away |

**Three objects, in order.** Keeping these apart is the key to reading this code.

| Object | Comes from | Lives | Job |
|---|---|---|---|
| `user` | What `authorize` returns | Only during sign-in | The person who just proved who they are |
| `token` | Managed by Auth.js | In the encrypted session cookie | Carries `id` and `role` between requests |
| `session` | Built from the token by the `session` callback | Each time `auth()` is called | What the app reads |

```text
authorize() → user → jwt() → token (cookie) → session() → auth() in the app
```

`jwt` runs again on later requests, but `user` is `undefined` then. That's why it has `if (user)`: the first run copies the values, and later runs keep what's already in the token.

**A session is only as current as its cookie.** `auth()` decrypts the cookie using `AUTH_SECRET` and doesn't query the database. So a user whose account was deleted, or whose role changed, keeps their old session until it expires. I saw this directly: a cookie I created for a user who didn't exist still made the navbar show them as signed in.

The cookie is encrypted, so the browser can't read or change the role inside it. Anyone holding `AUTH_SECRET` could create a valid session, so it has to stay private.

**Interview Q:** *What's the main downside of JWT sessions?*

You can't easily revoke one before it expires, and data inside it, like a role, can go stale. Sensitive actions may need a fresh database check.

**Interview Q:** *What's the difference between `user`, `token` and `session` in Auth.js?*

`user` exists only at sign-in and comes from `authorize`. `token` is what's stored in the cookie and persists between requests. `session` is the object your app reads, built from the token.

**Interview Q:** *Why does the `jwt` callback check `if (user)`?*

`user` is only passed at sign-in. On later calls it's `undefined`, so without the check the code would crash or overwrite the token's values.

**Interview Q:** *Why doesn't `session.user.role` exist without the `session` callback?*

Auth.js only exposes a few default fields. Custom values in the token have to be copied onto the session deliberately, which also stops private token data reaching the app by accident.

**Interview Q:** *What happens if you delete a user who is signed in with a JWT session?*

They stay signed in until the token expires, because nothing checks the database. To cut them off sooner, check the database on sensitive actions or use database sessions.

**Interview Q:** *Which `signIn` do you use in a client component?*

The one from `next-auth/react`. The one exported from the `NextAuth()` config is for server code.

### 4. Module augmentation: typing the session

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

**Interview Q:** *How do you add a property to a third-party library's types?*

Use `declare module` to extend its interface. Interfaces with the same name merge, which is one reason libraries expose interfaces for this.

### 5. Authentication vs authorization (401 vs 403)

- **Authentication:** who are you? Failing returns **401**.
- **Authorization:** are you allowed to do this? Failing returns **403**.

Hiding a button isn't security, because anyone can call the API directly. The check has to be on the server, so `getAuthorizedEvent` in `app/api/events/[slug]/route.ts` does it in three steps.

**Step 1: Authentication.** Nobody signed in means a 401.

```ts
// app/api/events/[slug]/route.ts
async function getAuthorizedEvent(params: Promise<{ slug: string }>): Promise<AuthorizedEventResult> {
  const session = await auth();
  if (!session?.user) {
    return { error: NextResponse.json({ message: 'Unauthorized' }, { status: 401 }) };
  }
```

**Step 2: Find the event.** An unknown slug means a 404.

```ts
  await connectDB();

  const { slug } = await params;
  const event = await Event.findOne({ slug: slug.trim().toLowerCase() });
  if (!event) {
    return { error: NextResponse.json({ message: 'Event not found' }, { status: 404 }) };
  }
```

**Step 3: Authorization.** Only an admin or the event's owner can continue. Anyone else gets a 403.

- `.toString()` is needed because `createdBy` is an `ObjectId`, and comparing it with a string using `===` is always false.
- **Bug I found:** older events had no `createdBy`, so `.toString()` threw and returned a 500 even for admins. `event.createdBy?.toString()` handles that case.

```ts
  const isAdmin = session.user.role === 'admin';
  const isOwner = event.createdBy.toString() === session.user.id;

  if (!isAdmin && !isOwner) {
    return { error: NextResponse.json({ message: 'You are not allowed to modify this event.' }, { status: 403 }) };
  }

  return { event, slug };
}
```

**Interview Q:** *What's the difference between 401 and 403?*

401 means the user isn't authenticated, so signing in might help. 403 means they're known but don't have permission.

### 6. Mass assignment: never trust the request body

Passing a request body straight into a database update lets the client change fields it should never touch.

**The bug.** An owner could send `{ "createdBy": "<someone-else's id>" }` and hand the event to another user, which also gets around the per-user event limit.

```ts
// The bug: PATCH /api/events/[slug]
const body = await request.json();
await Event.findOneAndUpdate({ slug }, { $set: body }, { new: true, runValidators: true });
```

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

**Interview Q:** *What is a mass assignment vulnerability?*

The server copies client input directly onto a model, so an attacker can set fields they shouldn't, such as `role` or an owner id. Prevent it with an allowlist of fields.

### 7. The sign-in form: `useAuthForm`, `signIn` and what happens after

One form handles both Sign in and Create account. A custom hook owns the selected tab and the submit logic, and `AuthPanel` only renders. The two tabs reach the server differently: sign-in goes through Auth.js, while sign-up calls my own API route, because Auth.js doesn't create accounts for a Credentials provider.

**A union type for the tab.** `mode` can only be one of two strings, so a typo like `"login"` is a compile error.

```ts
// hooks/useAuthForm.ts
export type AuthMode = "sign-in" | "sign-up";

export function useAuthForm(initialMode: AuthMode = "sign-in") {
const [mode, setMode] = useState<AuthMode>(initialMode);
const [isSubmitting, setIsSubmitting] = useState(false);
```

**Reading the inputs from the form.** The inputs are uncontrolled: the browser holds their values, and the code reads them only on submit, so typing doesn't cause re-renders. The `as` is needed because `namedItem` can return other types or `null`. It's only a compile-time claim, so a missing input would throw at runtime.

```ts
const readField = (form: HTMLFormElement, name: string) =>
  (form.elements.namedItem(name) as HTMLInputElement).value;
```

**Choosing what to run.** `preventDefault` stops the browser's normal submit, which would reload the page. `e.currentTarget` is the element the handler is attached to, which is the form. It's passed along right away because React clears `currentTarget` once the handler has finished, so reading it after an `await` would give `null`.

```ts
const handleSubmit = (e: React.SyntheticEvent<HTMLFormElement>) => {
  e.preventDefault();
  if (mode === "sign-up") submitSignUp(e.currentTarget);
  else submitSignIn(e.currentTarget);
};
```

**Signing in.** `"credentials"` picks the provider set up in `auth.ts` (section 2). `redirect: false` makes `signIn` return a result instead of navigating, so the page can show its own error and stay put. The email is trimmed but the password isn't, because spaces can be part of a password. The same `toastId` makes each later message replace the loading one instead of stacking.

```ts
const toastId = toast.loading("Signing in...");

const result = await signIn("credentials", {
  email: readField(form, "email").trim(),
  password: readField(form, "password"),
  redirect: false,
});

if (!result) throw new Error("Sign in returned no result."); // see the failure notes below
if (result.error) {
  toast.error("Invalid email or password.", { id: toastId });
  return;
}
```

The message is the same whether the email or the password was wrong, so the form doesn't reveal which emails have accounts.

**After signing in.** `router.push` goes to the home page. `router.refresh` asks the server to render again, because the navbar lives in the layout, and a layout isn't re-rendered on a normal client navigation. Without a refresh it could keep showing "Sign in".

```ts
toast.success("Welcome back!", { id: toastId });
router.push("/");
router.refresh();
```

**Creating an account.** `fetch` only rejects on network failures, so a 400 or 409 still resolves and `res.ok` has to be checked. `res.ok` is true for statuses 200 to 299. The server's message is shown when there is one.

```ts
const res = await fetch("/api/auth/signup", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name, email, password }),
});

const data = await res.json();

if (!res.ok) {
  toast.error(data.message || "Signup failed.", { id: toastId });
  return;
}

toast.success("Account created! Please sign in.", { id: toastId });
setMode("sign-in");
```

**Why every request is wrapped in `try/catch/finally`.** The main idea: a loading state turned on before an `await` has to be turned off on every path out, including the ones where the `await` throws. `try` holds the work, `catch` handles what went wrong, and `finally` does the cleanup no matter how the function ends.

```ts
// hooks/useAuthForm.ts
const submitSignUp = async (form: HTMLFormElement) => {
  setIsSubmitting(true);
  const toastId = toast.loading("Creating account...");

  try {
    const res = await fetch("/api/auth/signup", { /* ... */ });
    const data = await res.json();

    if (!res.ok) {
      toast.error(data.message || "Signup failed.", { id: toastId });
      return; // finally still runs
    }

    toast.success("Account created! Please sign in.", { id: toastId });
    setMode("sign-in");
  } catch (error) {
    console.error("Sign up failed:", error);
    toast.error("Something went wrong. Please try again.", { id: toastId });
  } finally {
    setIsSubmitting(false);
  }
};
```

**What happens without it.** My first version had no `try`, and `setIsSubmitting(false)` sat on the line after the `await`. When the `await` throws, the function stops right there, so that line never runs.

| Step | Without `try/catch/finally` | With it |
|---|---|---|
| The request throws | The function stops at the `await` | Jumps to `catch` |
| `setIsSubmitting(false)` | Never runs, so `isSubmitting` stays `true` | Runs in `finally` |
| Submit button | Disabled and stuck on "Creating..." | Enabled again |
| Loading toast | Stays forever, because a loading toast has no time limit | Replaced by the error message |
| What the user sees | A frozen form, with no explanation, until they refresh | A clear message, with their typed values still there |
| The error | An unhandled promise rejection in the console | Logged with context |

**What can throw here:**
- The network drops, so `fetch` rejects.
- A proxy returns an HTML error page, so `res.json()` throws on invalid JSON.

A 400 or 409 doesn't throw, which is why `res.ok` is still checked separately.

**Why `finally` and not a line after the `try`.** The `!res.ok` branch uses `return`, so code placed after the `try/catch` would be skipped on that path. `finally` runs after a `return` too, so the reset only has to be written once.

**The message is generic on purpose.** The raw `error.message` would be something like `Failed to fetch` or a JSON parsing error, which doesn't help the user. The detail goes to `console.error` instead.

**What `try/catch` can't do.** It only catches errors that are thrown. Auth.js's `signIn` catches a failure loading its provider list internally, sends the browser to `/api/auth/error` and returns nothing. So the code also checks for a missing result, otherwise `result?.error` would be `undefined` and the failure would count as a success.

```ts
if (!result) throw new Error("Sign in returned no result.");
if (result.error) {
```

I confirmed this by forcing each failure in a browser: a dropped request, a non-JSON reply, then a retry that succeeded.

**Interview Q:** *Why use `redirect: false` with `signIn`?*

By default it navigates after the attempt. With `false` you get the result back, so you can show an error on the same page and choose where to go on success.

**Interview Q:** *Why call `router.refresh()` after signing in?*

Layouts keep their rendered output across client navigations, so a Server Component like the navbar would still show the signed-out state. `refresh` fetches fresh Server Component output without losing client state.

**Interview Q:** *Does `fetch` throw on a 500 response?*

No. It only rejects when the request can't complete, such as a network failure. HTTP errors resolve normally, so check `res.ok`.

**Interview Q:** *Why show one message for a wrong email or a wrong password?*

Different messages would let someone test which emails are registered, which is called user enumeration.

**Interview Q:** *Controlled vs uncontrolled inputs?*

A controlled input keeps its value in React state and re-renders on each keystroke, which suits live validation. An uncontrolled input keeps its value in the DOM and you read it when needed, which is simpler for a form that's only read on submit.

**Interview Q:** *Where should you turn off a loading state after a request?*

In `finally`, so it runs after both success and failure.

**Interview Q:** *What happens if an `await` throws and there's no `try/catch`?*

The function stops at that line and returns a rejected promise. Nothing after it runs, so cleanup like turning off a loading state is skipped and the interface can stay stuck.

**Interview Q:** *Does `finally` run if the `try` block returns?*

Yes. It runs after `return`, after a thrown error, and after normal completion, which makes it the right place for cleanup.

**Interview Q:** *Will a React error boundary catch an error from an async submit handler?*

No. Error boundaries catch errors thrown while rendering. Errors in event handlers and async code have to be caught in that code.

**Interview Q:** *Can `try/catch` catch every failure from a library call?*

Only errors the library actually throws. If it swallows an error and returns nothing or navigates away, you have to check the return value too.

**Interview Q:** *Why not show `error.message` to the user?*

It's written for developers and can reveal internal details. Log it, and show a message the user can act on.

**Interview Q:** *Is `disabled={isSubmitting}` enough to prevent duplicate submissions?*

It's a good interface guard, but it doesn't protect the server. Duplicates can still arrive from other sources, so uniqueness has to be enforced there too, such as with a unique index.

---

## Project decisions

These came from questions I asked while building StackMeet.

### 8. Guest bookings: an unverified email doesn't prove ownership

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

**Interview Q:** *A user signs up with an email that already has guest orders. Should you link them automatically?*

Only after verifying the email. Otherwise, signing up with someone else's address gives access to their data.
