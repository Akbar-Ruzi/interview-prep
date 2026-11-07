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
- [Project decisions](#project-decisions)
  7. Guest bookings: an unverified email doesn't prove ownership

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

**Interview Q:** *How should passwords be stored?* Never as plain text. Store a salted hash from a slow algorithm designed for passwords, such as bcrypt or Argon2, and verify logins with its compare function.

### 2. Credentials sign-in with `authorize`

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

**Interview Q:** *Why not tell users which part of the login was wrong?* It would let attackers discover which emails are registered, and then target those accounts.

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

**Interview Q:** *What's the main downside of JWT sessions?* You can't easily revoke one before it expires, and data inside it, like a role, can go stale. Sensitive actions may need a fresh database check.

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

**Interview Q:** *How do you add a property to a third-party library's types?* Use `declare module` to extend its interface. Interfaces with the same name merge, which is one reason libraries expose interfaces for this.

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

**Interview Q:** *What's the difference between 401 and 403?* 401 means the user isn't authenticated, so signing in might help. 403 means they're known but don't have permission.

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

**Interview Q:** *What is a mass assignment vulnerability?* The server copies client input directly onto a model, so an attacker can set fields they shouldn't, such as `role` or an owner id. Prevent it with an allowlist of fields.

---

## Project decisions

These came from questions I asked while building StackMeet.

### 7. Guest bookings: an unverified email doesn't prove ownership

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
