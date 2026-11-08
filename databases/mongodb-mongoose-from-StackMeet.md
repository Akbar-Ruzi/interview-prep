# Interview Notes: MongoDB and Mongoose

> Every snippet is from my project (StackMeet, a developer events app: Next.js 16, React 19, TypeScript, Mongoose 8, Auth.js v5), shortened where needed.
> Snippets marked **Planned fix** show the fix I would make. They aren't in the project yet.
> For each concept: what it is, the project code, and a likely interview question.
> Other notes from this project: [Next.js API and server code](../backend/nextjs-api-from-StackMeet.md), [Next.js rendering and config](../frontend/nextjs-from-StackMeet.md) and [Authentication and authorization](../auth/auth-from-StackMeet.md).

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
  10. `userId?`: an optional field in the type and the schema
  11. `createBooking`: where the data actually gets saved
  12. Building a query from optional filters: `$and`, `$or`, `$ne`, `$in`
  13. From query to page data: lazy queries, `limit` and plain ids
- [Project decisions](#project-decisions)
  14. Protecting a free database: why one limit isn't enough

---

## MongoDB and Mongoose

### 1. Schemas, validation and TypeScript types

MongoDB doesn't enforce a structure by default, so a Mongoose schema adds one at the application level. The TypeScript interface checks types at compile time, while the schema validates data at runtime. You need both.

**The plain data types.** Splitting `EventFields` from `IEvent` gives a plain data type for props, separate from the full database document.

```ts
// database/event.model.ts
export interface AgendaItem {
  time: string;
  item: string;
}

export interface EventFields {
  title: string;
  mode: 'online' | 'in-person' | 'hybrid';
  agenda?: AgendaItem[];
  // ...
}
```

**The stored document type.** This is the fields plus what MongoDB and Mongoose add.

- `IEvent extends EventFields, Document` combines both types, because an interface can extend more than one. `IEvent` gets the event fields, plus Mongoose members such as `_id` and `save()`.
- `Document` here is imported from `mongoose`. It isn't the browser's DOM `Document`.

```ts
interface IEvent extends EventFields, Document {
  createdAt: Date;
  updatedAt: Date;
  createdBy: Types.ObjectId; // reference to the User who created the event
}
```

**The schema: a simple field.** `title` is required, trimmed and limited to 100 characters. The steps below add the other fields to this same schema.

```ts
const EventSchema = new Schema<IEvent>(
  {
    title: {
      type: String,
      required: [true, 'Title is required'],
      trim: true,
      maxlength: [100, 'Title cannot exceed 100 characters'],
    },
```

**A fixed list of values: `mode`.**

- `enum` limits a string field to a fixed list. `values` is the list, and `message` is the error returned when a value isn't in it. A missing value passes `enum`, so `mode` also needs `required`.
- The `mode` union type and the `enum` list look like duplicates, but the union checks your code at compile time, and `enum` checks real data at runtime. They must be kept in sync by hand here.

```ts
    mode: {
      type: String,
      required: [true, 'Mode is required'],
      enum: {
        values: ['online', 'in-person', 'hybrid'],
        message: 'Mode must be either online, in-person, or hybrid',
      },
    },
```

**A list of subdocuments: `agenda`.**

- `type: [{ ... }]` means "a list of", and the object inside describes one item, such as `{ time: '10:00', item: 'Introduction' }`.
- Mongoose gives every subdocument its own `_id` by default. `_id: false` turns that off, because agenda items are never looked up on their own.
- Rules inside the object apply to each item, while rules beside `type`, such as a `validate`, would apply to the whole list. A missing time in the first item fails at the path `agenda.0.time`, so the error points to the exact item.
- `trim: true` removes extra spaces when the value is set, so `' 10:00 '` is saved as `'10:00'`.
- `agenda` is optional, so it has no rule for the whole list. If it had to contain at least one item, `required` wouldn't be enough, because an empty array still passes. It would need a custom validator: `validate: { validator: (v: AgendaItem[]) => v.length > 0, message: 'At least one agenda item is required' }`.

```ts
    agenda: {
      type: [
        {
          _id: false,
          time: {
            type: String,
            required: [true, 'Agenda item time is required'],
            trim: true,
          },
          item: {
            type: String,
            required: [true, 'Agenda item text is required'],
            trim: true,
          },
        },
      ],
    },
```

**A list of strings: `tags`.** `type: [String]` means a list of strings. It has no other rules, so it's optional, and Mongoose stores `[]` when it's left out. The short form `tags: [String]` means the same thing. The longer form is only needed to add options like `validate`.

```ts
    tags: {
      type: [String],
    },
```

**A reference to another model: `createdBy`.** `ref: 'User'` enables `.populate()`. It doesn't create a foreign key, so MongoDB won't stop a reference to a deleted user.

```ts
    createdBy: { type: Schema.Types.ObjectId, ref: 'User', required: true, index: true },
  },
```

**Timestamps.** The interface only describes the shape and is removed at compile time. `timestamps: true` is what actually sets `createdAt` and `updatedAt`.

```ts
  { timestamps: true } // adds createdAt and updatedAt automatically
);
```

**Conditional `required` with `this`:** `required` can take a function instead of `true`. Here, `venue` and `location` are required only when the event has a physical place.

First, the function that decides:

- The function takes no real arguments. Mongoose calls it during validation with `this` set to the event being validated, so `this.mode` is that event's format.
- `this: IEvent` is a TypeScript-only annotation that tells the compiler what `this` is. It must come first in the parameter list and is removed at compile time, so the function is called with no arguments.
- It must be a regular function. An arrow function doesn't get its own `this`, and TypeScript doesn't allow a `this` parameter on one.

```ts
// database/event.model.ts
function needsPhysicalPlace(this: IEvent): boolean {
  return this.mode === 'in-person' || this.mode === 'hybrid';
}
```

Then the field passes that function to `required`:

```ts
venue: {
  type: String,
  required: [needsPhysicalPlace, 'Venue is required for in-person and hybrid events'],
  trim: true,
},
```

**Interview Q:** *What is `this: IEvent` in a function's parameter list?* It's a fake parameter that only declares the type of `this`. Without it, TypeScript can't tell what `this` is, and `strict` mode reports an error. It doesn't exist in the compiled JavaScript.

**Interview Q:** *Is `this` always the document in a Mongoose validator?* No. On `save()` and `create()` it's the document. With `findOneAndUpdate` and `runValidators: true`, `this` is the query, so `this.mode` is `undefined` and a rule like this one won't apply.

**Interview Q:** *TypeScript already has types, so why do you need a schema?* TypeScript types are removed at compile time, so they can't check data that arrives at runtime, such as a request body. The schema does that check before anything is saved.

**Interview Q:** *Is `extends Document` the recommended way to type a Mongoose model?* No. Mongoose's TypeScript docs recommend against it. They suggest a plain interface for the fields, such as `Schema<EventFields>`, and `HydratedDocument<EventFields>` where you need the full document type.

**Interview Q:** *With `type: [String]`, what happens if you save `tags: [123]`?* It's saved as `['123']`, because Mongoose converts simple values to the schema's type before validating. Values it can't convert, such as an object or a nested array, fail validation.

**Interview Q:** *What does `_id: false` do on a subdocument, and when would you use it?* It stops Mongoose adding an `_id` to each item in the list. Use it when items are only ever read as part of their parent. Keep the `_id` if you need to find, update or remove one specific item, such as a comment in a list of comments.

**Interview Q:** *Does a Mongoose `enum` always stop invalid values?* No. It runs when Mongoose validates, such as on `save()` and `create()`. `findOneAndUpdate` skips it unless you pass `runValidators: true`, and MongoDB itself doesn't know about the rule, so a write that bypasses Mongoose isn't checked.

### 2. `models.X || model()`: avoiding `OverwriteModelError`

Mongoose keeps a registry of compiled models and throws `OverwriteModelError` if the same name is compiled twice. Next.js hot reload re-runs modules in development, so this line reuses the existing model when there is one.

```ts
// database/event.model.ts
const Event = models.Event || model<IEvent>('Event', EventSchema);
export default Event;
```

**Interview Q:** *Why not just call `model('Event', schema)`?* Because any module that runs again, such as after a hot reload, would try to register the model twice and throw.

### 3. Caching the database connection

Opening a MongoDB connection is slow and connections are limited. Without a cache, every hot reload or request could open a new one until the limit is reached.

**Keep the cache on `global`.** `global` survives hot reloads, which reset module-level variables. A serverless instance can also reuse it across requests while it stays warm.

```ts
// lib/mongodb.ts
const cached: MongooseCache = global.mongoose || { conn: null, promise: null };

if (!global.mongoose) {
  global.mongoose = cached;
}
```

**Reuse an open connection.** If one already exists, return it straight away. The next two steps also run inside `connectDB`.

```ts
async function connectDB(): Promise<typeof mongoose> {
  if (cached.conn) return cached.conn;
```

**Start connecting only once.** The promise is cached as well, so if two requests arrive during connection, both await the same promise instead of opening two connections. With `bufferCommands: false`, queries fail right away if there's no connection, rather than waiting in a queue.

```ts
  if (!cached.promise) {
    if (!MONGODB_URI) throw new Error('Please define the MONGODB_URI environment variable');

    cached.promise = mongoose.connect(MONGODB_URI, { bufferCommands: false });
  }
```

**Wait for the connection, and reset on failure.** Without the reset, every later request would keep awaiting the same rejected promise, and the app could never reconnect.

```ts
  try {
    cached.conn = await cached.promise;
  } catch (error) {
    cached.promise = null; // allow a retry after a failure
    throw error;
  }

  return cached.conn;
}
```

**Interview Q:** *How do you manage database connections in serverless?* Cache one connection per instance on `global`, and cache the in-progress promise so concurrent requests share it. Each instance still holds its own connection, so many instances can still use up the limit.

### 4. Pre-save hooks: generating a slug

Middleware (hooks) runs logic automatically around an operation. This hook keeps derived data correct wherever the event is saved.

**The hook.**

- It uses a regular `function`, not an arrow function, because the hook needs `this` to be the document.
- `isModified` avoids unnecessary work when the field didn't change.
- **Design trade-off:** regenerating the slug when a title changes breaks links people have already shared. I decided to keep the original slug on edit, so `isNew` limits it to new events.

```ts
// database/event.model.ts
EventSchema.pre('save', function (next) {
  if (this.isNew) {
    this.slug = generateSlug(this.title);
  }
  if (this.isModified('date')) {
    this.date = normalizeDate(this.date);
  }
  next();
});
```

**The slug helper.** This turns a title into a URL-friendly string, one step per line.

```ts
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

**Locking a field with `immutable`:** this option lets a field be set when the document is created, then ignores later changes to it.

- A save hook alone can't protect the slug, because `findOneAndUpdate` skips save hooks. `immutable` also covers that path: Mongoose removes the field from the update before it reaches MongoDB.
- By default, the change is ignored silently and no error is raised. A request that sets both `slug` and `title` still succeeds, and only `title` changes.
- The rule lives in Mongoose, not MongoDB, so a write that bypasses Mongoose can still change the field.

```ts
// database/event.model.ts
slug: {
  type: String,
  immutable: true,
  unique: true,
  lowercase: true,
  trim: true,
},
```

**Interview Q:** *What does `immutable: true` do in a Mongoose schema?* It allows the field to be set on a new document, then ignores later changes, both on `save()` and in updates like `findOneAndUpdate`. In my project, it stops an event's slug changing, so shared links keep working.

**Interview Q:** *Why can't a Mongoose hook be an arrow function?* Arrow functions don't have their own `this`, and document middleware uses `this` to access the document.

### 5. `findOneAndUpdate` skips save hooks

This was a real bug in my project.

**The bug.** `pre('save')` is **document** middleware, while `findOneAndUpdate` is **query** middleware. It updates MongoDB directly and never calls `save()`, so the date normalization in the hook doesn't run.

Update validators are off by default, so `runValidators: true` is needed for them to run at all. `findOneAndUpdate` is one atomic operation, which suits counters and other concurrent changes.

```ts
// app/api/events/[slug]/route.ts: the bug
const updatedEvent = await Event.findOneAndUpdate(
  { slug: result.slug },
  { $set: body },
  { new: true, runValidators: true }
);
```

**Planned fix:** load the document, change it and call `save()`. This takes two database calls, and it's a good default when hooks hold business logic.

```ts
const event = await Event.findOne({ slug });
event.set(allowedFields); // only the fields users may edit
await event.save();       // runs validators and pre('save')
```

**Interview Q:** *When would you use `save()` over `findOneAndUpdate`?* Use `save()` when you need document hooks and full validation. Use `findOneAndUpdate` when the update has to be atomic and doesn't depend on hooks.

### 6. Indexes: unique, compound and redundant

An index lets MongoDB find documents without scanning the whole collection. The cost is extra storage and slightly slower writes.

- **Prefix rule:** a compound index can serve queries on its leading fields. `{ eventId: 1, createdAt: -1 }` already covers queries on `eventId`, so the first index below is redundant.
- **Field order matters:** a common guideline is Equality, then Sort, then Range (ESR).

```ts
// database/booking.model.ts
BookingSchema.index({ eventId: 1 });
BookingSchema.index({ eventId: 1, createdAt: -1 }); // an event's bookings, newest first
BookingSchema.index({ email: 1 });                  // a user's bookings
BookingSchema.index({ eventId: 1, email: 1 }, { unique: true, name: 'uniq_event_email' });
```

**Duplicate declarations:** `unique: true` on a field already creates an index. Adding `schema.index()` for the same field triggered this warning in my build:

```text
[MONGOOSE] Warning: Duplicate schema index on {"email":1} found.
```

**Interview Q:** *Does an index on `(a, b)` help a query that only filters on `b`?* Generally no. It helps queries on `a`, or on `a` and `b`, because the index is sorted by `a` first.

### 7. A unique index beats "check, then insert"

Checking first and then inserting has a race condition, because two requests can both pass the check. Only the database can guarantee uniqueness.

**Add the unique index.** This allows one booking per email for each event.

```ts
// Booking schema
BookingSchema.index({ eventId: 1, email: 1 }, { unique: true, name: 'uniq_event_email' });
```

**Insert, and handle the duplicate error.** There's no check first. The code tries the insert, and `11000` means the index rejected a duplicate.

```ts
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

My sign-up route shows the problem. The unique index still blocks the duplicate, so the data stays correct. But the second request gets a 500 instead of a 409, so it should also catch error `11000`.

```ts
// app/api/auth/signup/route.ts
const existingUser = await User.findOne({ email }).lean();
if (existingUser) {
  return NextResponse.json({ message: 'Email already exists' }, { status: 409 });
}
// A second request can reach this line at the same time
const createdUser = await User.create({ name, email, password: hashedPassword, role: 'user' });
```

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

**Claim a slot.** MongoDB writes to a single document are atomic, so parallel requests can't both pass the `$lt` check for the last slot.

```ts
const user = await User.findOneAndUpdate(
  { _id: userId, eventCount: { $lt: MAX_EVENTS } }, // only matches if under the limit
  { $inc: { eventCount: 1 } },
  { new: true }
);
if (!user) return limitReachedResponse();
```

**Create the event, and give the slot back on failure.** The counter and the event are separate writes, so a failed create has to undo the increment. A transaction is the stricter alternative.

```ts
try {
  await Event.create(eventPayload);
} catch (error) {
  await User.updateOne({ _id: userId }, { $inc: { eventCount: -1 } }); // give the slot back
  throw error;
}
```

**Interview Q:** *How do you enforce a limit safely under concurrency?* Combine the check and the update into one atomic operation, such as a conditional `$inc`, rather than reading first and writing later.

### 9. `.lean()`

`.lean()` returns plain JavaScript objects instead of full Mongoose documents, so queries use less memory and CPU. Lean results don't have `save()`, virtuals or getters, so use it for read-only data, not for documents you'll change.

```ts
const event = await Event.findOne({ slug: sanitizedSlug }).lean();
```

**Interview Q:** *When shouldn't you use `.lean()`?* When you need to change and `save()` the document, or when you rely on virtuals, getters or document methods.

### 10. `userId?`: an optional field in the type and the schema

Guests can book without an account, so they have no user ID. If `userId` were required, every guest booking would fail. The type and the schema have to agree, because they do different jobs.

**The type.** The `?` makes TypeScript remind you to handle the empty case wherever you use it. It only exists at compile time.

```ts
// database/booking.model.ts
export interface IBooking extends Document {
  userId?: Types.ObjectId; // Set when a signed-in user books; empty for guests
  name: string;
  email: string;
}
```

**The schema.** The missing `required` rule is what allows an empty value at runtime.

```ts
const BookingSchema = new Schema<IBooking>({
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    index: true, // no `required`, unlike name and email
  },
});
```

**Interview Q:** *What's the difference between `userId?: string` and `userId: string | undefined`?* With `?`, the property can be left out entirely. With `| undefined`, the property must be present, although its value can be `undefined`.

### 11. `createBooking`: where the data actually gets saved

The save is the `Booking.create()` call. Everything before it makes sure the data is safe to save.

`booking.action.ts` holds the booking functions that run on the server. The `'use server'` line lets the form call them directly, without an API route.

**The outline.** Steps 1 to 4 run inside the `try`, and step 5 runs in the `catch`.

```ts
// lib/actions/booking.action.ts (shortened)
export const createBooking = async ({ eventId, email, name, phone }: CreateBookingParams) => {
  try {
    // Steps 1 to 4
  } catch (e) {
    // Step 5
  }
};
```

**Step 1: Clean the input.**

```ts
const cleanName = cleanText(name);
const normalizedEmail = cleanText(email).toLowerCase();
const cleanPhone = cleanText(phone);
```

**Step 2: Validate it, and return a message the form can show.**

```ts
const validationError = getValidationError(cleanName, normalizedEmail, cleanPhone);
if (validationError) {
  return { success: false, message: validationError } satisfies CreateBookingResult;
}
```

**Step 3: Connect to the database and find out who is booking.**

```ts
await connectDB();
const session = await auth();
```

**Step 4: Save.** Mongoose checks the schema and runs the pre-save hook, then MongoDB stores the booking.

```ts
await Booking.create({
  eventId,
  userId: session?.user?.id,
  name: cleanName,
  email: normalizedEmail,
  phone: cleanPhone || undefined,
});
return { success: true } satisfies CreateBookingResult;
```

**Step 5: Handle failures.** `11000` is MongoDB's duplicate key error, from the unique index on `eventId` + `email`. Anything else gets a general message.

```ts
if (typeof e === 'object' && e !== null && 'code' in e && (e as { code?: number }).code === 11000) {
  return { success: false, message: 'This email is already registered for this event.' } satisfies CreateBookingResult;
}
console.log('Create booking failed:', e);
return { success: false, message: 'Unable to create booking right now. Please try again.' } satisfies CreateBookingResult;
```

**Interview Q:** *Why get `userId` from `auth()` instead of passing it from the form?* A Server Action is a public endpoint, so a caller could send any ID they like. The session is checked on the server, so it can't be faked that way.

**Interview Q:** *Why is `e` typed as `unknown` in a `catch` block?* Anything can be thrown in JavaScript, not just an `Error`, so you have to check its shape before reading a property like `code`. That's what the `typeof` and `'code' in e` checks do. In strict TypeScript projects, `catch` variables default to `unknown`.

### 12. Building a query from optional filters: `$and`, `$or`, `$ne`, `$in`

The home search has optional boxes, so the query can't be written as one fixed object. The function starts with the condition that always applies, then adds one condition for each box the user filled in. It's the database version of the array filter in [Next.js notes #3](../frontend/nextjs-from-StackMeet.md#3-search-filter-logic-normalize-input-or-inside-a-filter-and-between-filters). Filtering in the database means `limit` applies after the filters.

**The start.** Only upcoming events ever qualify. `$gte` works on these date strings because `YYYY-MM-DD` sorts in date order ([Next.js notes #4](../frontend/nextjs-from-StackMeet.md#4-upcoming-events-compare-date-strings-sort-with-localecompare)).

```ts
// lib/data/events.ts
function buildUpcomingFilter({ q, where, mode }: UpcomingEventsOptions): FilterQuery<EventFields> {
  const conditions: FilterQuery<EventFields>[] = [{ date: { $gte: today() } }];
```

**The search box.** `$or` means the title or a tag can match. When a field holds an array, like `tags`, MongoDB matches if any element matches. The `"i"` flag ignores case.

```ts
  const what = clean(q);
  if (what) {
    const pattern = new RegExp(escapeRegex(what), "i");
    conditions.push({ $or: [{ title: pattern }, { tags: pattern }] });
  }
```

**The place box and the mode.** The word `online` checks the format, and anything else checks the city.

```ts
  const place = clean(where);
  if (place === "online") conditions.push({ mode: { $ne: "in-person" } });
  else if (place) conditions.push({ location: new RegExp(escapeRegex(place), "i") });

  if (mode) conditions.push({ mode });
```

**Combining them.** `$and` means every condition must pass.

```ts
  return { $and: conditions };
}
```

For `{ q: "react", where: "online" }` the result is:

```ts
{ $and: [
  { date: { $gte: "2026-11-08" } },
  { $or: [{ title: /react/i }, { tags: /react/i }] },
  { mode: { $ne: "in-person" } },
] }
```

**Escaping user input in a regex.** Without this, characters a user types are treated as regex syntax. `react.js` would also match `reactXjs`, and `(` would throw an invalid regex error. `$&` in the replacement stands for the matched character, so each special character gets a backslash in front of it.

```ts
const escapeRegex = (value: string) => value.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
```

**`$in` and `$ne` together.** Similar events are the ones sharing at least one tag, excluding the event itself.

```ts
const similar = await Event.find({ _id: { $ne: event._id }, tags: { $in: event.tags } })
  .select(PUBLIC_FIELDS)
  .lean<EventDoc[]>();
```

| Operator | Meaning |
|---|---|
| `$gte` | Greater than or equal to |
| `$ne` | Not equal. Also matches documents where the field is missing |
| `$in` | Equals any value in the list. On an array field, any element can match |
| `$or` | At least one condition passes |
| `$and` | Every condition passes |

**Limitations to know.** A case-insensitive regex that isn't anchored to the start can't use an index efficiently, so MongoDB checks many documents. That's fine for a small collection, but a large one needs a text or search index. Also limit how long the search input can be.

**Interview Q:** *How do you build a MongoDB query from optional filters?* Start an array with the conditions that always apply, push one condition for each filter that has a value, and return `{ $and: conditions }`.

**Interview Q:** *Why escape user input before `new RegExp()`?* Otherwise the user's text is treated as a pattern. It can match the wrong things, throw on invalid syntax, or run a slow pattern against the database.

**Interview Q:** *What's the catch with `{ field: { $ne: value } }`?* It also matches documents that don't have the field. If that matters, add `$exists: true` or make the field required in the schema.

### 13. From query to page data: lazy queries, `limit` and plain ids

`Event.find()` doesn't run anything. It builds a query object, and each chained call adds to it.

**Build the query.** `1` means ascending, so this sorts by date, then by time within a date. The `limit` is added only if the caller asked for one.

```ts
// lib/data/events.ts
await connectDB();
const query = Event.find(buildUpcomingFilter(options)).select(PUBLIC_FIELDS).sort({ date: 1, time: 1 });
if (limit) query.limit(limit);
```

**Run it.** `await` is what sends the query to MongoDB. `.lean()` returns plain objects ([#9](#9-lean)), and `.map(toRecord)` converts each one.

```ts
return (await query.lean<EventDoc[]>()).map(toRecord);
```

**Limit in the database, not in JavaScript.** The first version loads every match into server memory and throws most away. The second makes MongoDB return only five.

```ts
(await Event.find(filter)).slice(0, 5);  // loads everything, keeps 5
await Event.find(filter).limit(5);       // only 5 leave the database
```

**Convert ids to strings.** Even with `.lean()`, `_id` and `createdBy` are `ObjectId` instances. React can only pass plain data to Client Components, so a class instance like `ObjectId` is rejected. `...doc` copies every field, and the two lines after it replace the ids.

```ts
const toRecord = (doc: EventDoc): EventRecord => ({
  ...doc,
  _id: doc._id.toString(),
  createdBy: doc.createdBy.toString(),
});
```

**Planned fix: list allowed fields instead of excluded ones.** `"-createdAt -updatedAt -__v"` hides three fields and returns everything else, so a sensitive field added later would be sent to pages automatically. Listing the allowed fields means a new field stays hidden until someone chooses to expose it.

```ts
const PUBLIC_FIELDS = "title slug description overview image venue location date time mode audience agenda organizer tags createdBy";
```

**Interview Q:** *When does a Mongoose query actually run?* When it's awaited, or when `.then()` or `.exec()` is called. `find`, `select`, `sort` and `limit` only build it, which is why a condition can add `.limit()` before it runs.

**Interview Q:** *Why use `.limit()` instead of `.slice()`?* `.limit()` stops the extra documents from leaving the database, which saves memory and network time. It also has to come with a `.sort()`, or which documents you get isn't guaranteed.

**Interview Q:** *Why convert `ObjectId` to a string before passing it to a Client Component?* Props crossing from server to client must be serializable, and React rejects class instances like `ObjectId`. A string works everywhere.

**Interview Q:** *Why is a field allowlist safer than an exclusion list?* With an exclusion list, any new field is exposed by default. With an allowlist, it's hidden by default.

---

## Project decisions

These came from questions I asked while building StackMeet.

### 14. Protecting a free database: why one limit isn't enough

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
