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
- [Questions I asked about my code](#questions-i-asked-about-my-code)
  4. Constants in a `'use server'` file

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

### 3. Uploading a file: wrapping a callback API in a Promise

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

---

## Questions I asked about my code

### 4. Constants in a `'use server'` file

*My question about these constants: "are they going to be used somewhere?"*

```ts
// lib/actions/booking.action.ts
const MAX_NAME_LENGTH = 50;
const MIN_PHONE_DIGITS = 7;
const MAX_PHONE_DIGITS = 15;
const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const PHONE_REGEX = /^\+?[\d\s().-]+$/;
```

They're used by `getValidationError` in the same file. Naming them keeps the rules in one place and explains what a bare `50` or `15` means.

They can't be exported from this file, because a `'use server'` file can only export async functions. **Planned fix:** when the new booking form needs the same rules, move them into a small shared file that both the form and the server import.

**Interview Q:** *Why can a `'use server'` file only export async functions?* Each export becomes a function the browser can call over the network, and a network call is always asynchronous. A plain value like a number or regex can't be called that way.

**Interview Q:** *Why validate on both the client and the server?* Client validation gives fast feedback, but it's only for convenience, because anyone can skip it and call the server directly. Server validation is what actually protects the data.
