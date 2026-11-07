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
// components/BookEvent.tsx (client component)
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

Finally, check the password's length. bcrypt has a practical 72-byte limit, so the route enforces a limit at input. **Known issue:** this check counts characters, not bytes (see [Hashing passwords with bcrypt](../auth/auth-from-StackMeet.md)).

```ts
if (password.length < 8 || password.length > 72) {
  return NextResponse.json(
    { message: "Password must be between 8 and 72 characters" },
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
