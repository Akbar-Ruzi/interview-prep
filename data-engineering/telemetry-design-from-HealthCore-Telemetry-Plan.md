# Interview Notes: Telemetry Design and Data Pipelines

> Every snippet is from my project (HealthCore Telemetry Plan: a telemetry design for a 12-clinic US/UK clinical-supply inventory system).
> For each concept: what it is, the project snippet, and a likely interview question.

## Contents

- [Telemetry fundamentals](#telemetry-fundamentals)
  1. Telemetry vs logs vs metrics
  2. The golden rule: hypothesis → decision
  3. Mandatory metrics vs identified opportunities
  4. Instrumentation points (touchpoints)
  4b. Contract first, infrastructure later (data contracts)
  4c. Emit business events after the commit, not on submit
  4d. Edge-triggered vs level-triggered alerts
  4e. Where to emit: backend vs frontend
- [Event design](#event-design)
  5. The Event Envelope
  6. Idempotency and deduplication with `eventId`
  7. Correlation with `requestId`
  8. Schema versioning with `schemaVersion`
  9. `entity_action` naming taxonomy
  10. Property allowlist
- [JSON Schema](#json-schema)
  11. JSON Schema draft-07 basics
  12. `allOf` + `$ref`: reusing the envelope
  13. `const` and `additionalProperties: false`
  14. Keeping the docs and the schema in sync
- [Delivery strategy](#delivery-strategy)
  15. Stream vs batch processing
  16. Throttle vs debounce for high-frequency events
- [Privacy and compliance](#privacy-and-compliance)
  17. PII vs PHI, HIPAA and UK GDPR
  18. Data minimisation, anonymisation, and pseudonymisation
  19. Risks and exclusions: deliberate scope cuts

---

## Telemetry fundamentals

### 1. Telemetry vs logs vs metrics

- **Logs**: free-text or semi-structured records of what happened, mainly for debugging.
- **Metrics**: numeric aggregates over time (counts, rates, p95 latency).
- **Telemetry events**: structured, schema-defined records of a business or technical fact, designed upfront to answer specific questions. Metrics are often *derived* from events.

```md
<!-- plan.md -->
The deliverable is **design documentation, not code**. It must be precise enough
that another developer can instrument the existing system without asking questions.
```

**Interview Q:** *Why design a telemetry plan before writing code?* Instrumenting ad hoc produces inconsistent names, missing fields, and leaked sensitive data. A plan fixes the event names, the payload, and the reason for each event before anyone writes `track()`.

### 2. The golden rule: hypothesis → decision

Every event must justify itself with a question it answers and a decision it enables. If you can't complete the sentence, you don't capture the event.

```md
<!-- plan.md -->
> "We capture `[event_type]` because we need to know `[hypothesis]`,
> which allows us to make the decision `[concrete decision]`."
```

Example: we capture `stock_threshold_triggered` because we need to know how often a clinic runs short of a critical supply, which lets us prioritise urgent restocking and escalate to Clinical Operations.

**Interview Q:** *How do you avoid collecting "just in case" data?* Tie every event to a decision. Data without a decision costs storage and money, adds privacy risk, and creates noise. Anything that fails the test goes into a "discarded" list with the reason.

### 3. Mandatory metrics vs identified opportunities

- **Mandatory**: required by the business (here, the BRIEFING). Names must match exactly.
- **Identified opportunity**: proposed by the engineer after exploring the system (auth, performance, errors, navigation).

```md
<!-- plan.md -->
| `inbound_order_created`      | Inventory      | mandatory  | BRIEFING |
| `login_failed`               | Authentication | identified | Solution |
| `api_latency_recorded`       | Performance    | identified | README + Solution |
```

**Interview Q:** *Why not just implement the metrics stakeholders asked for?* Stakeholders ask for business metrics. They rarely think of operational signals like auth failures, latency, or frontend errors, which often matter just as much. Labeling the two makes it clear what's required and what's your proposal.

### 4. Instrumentation points (touchpoints)

The exact place in a user flow or system path where an event is emitted. You find them by walking the flow end to end.

```md
<!-- telemetry-plan.md -->
Login → Open inventory → (Direct stock edit attempt) → Submit order → Validation → Order saved → Stock recalculated

| # | Step              | What can happen                                         | Event fired                  | Emitted by |
| 4 | Direct stock edit | The user tries to change stock directly; it is rejected | `direct_stock_edit_rejected` | Backend    |
| 5 | Submit order      | The server rejects the order (missing field, ...)       | `order_validation_failed`    | Backend    |
| 6 | Order saved       | The order is saved successfully and stock is updated    | `inbound_order_created` / `outbound_order_created` | Backend |

Each order submission ends in exactly one outcome: `order_validation_failed` (nothing saved)
or `inbound_order_created` / `outbound_order_created` (saved).
```

Design each step's outcomes to be **mutually exclusive**: one attempt = one event. Then "failure rate = failed / (failed + created)" is a simple, correct ratio.

**Interview Q:** *Should you track only successes?* No. Failures and rejections (validation errors, blocked actions) often tell you more: where users get stuck, or who is trying to bypass controls.

### 4e. Where to emit: backend vs frontend

- **Backend**: the source of truth for anything that changes data or enforces a rule (orders saved, stock edits rejected, logins failed). It can't be skipped by calling the API directly, and it isn't lost to ad blockers or closed tabs.
- **Frontend**: only for things the server can't see (screens viewed, page load time, browser errors).

```md
<!-- telemetry-plan.md -->
The backend is the source of truth: a frontend that hides the option is not enough,
because the API can be called directly.
```

**Interview Q:** *Why not fire `order_created` from the frontend when the user clicks Save?* The click doesn't mean the order was saved. Validation or the DB can still fail, retries can double-count, and client events get lost. Emit from the backend after the commit; keep frontend events for UI behaviour.

### 4b. Contract first, infrastructure later (data contracts)

A **data contract** is the agreed shape and meaning of the data a producer emits: event names, fields, types, allowed values, privacy rules, and versioning. Consumers (dashboards, reports, alerts) build against it. You define the contract first and pick infrastructure (queues, Kafka, warehouse) later.

```md
<!-- plan.md -->
The deliverable is **design documentation, not code**.
...
This telemetry will later feed the network operations dashboard, Dr. Okonkwo's
monthly executive report, and Claire's compliance alerts.
```

Future consumers shape the contract (every event carries `clinic_id` and `country` so it can be aggregated by clinic and country), but they aren't built now.

**Interview Q:** *Why design the contract before choosing the pipeline?* The contract is the expensive part to change: once many producers emit events and many consumers read them, renaming a field breaks everyone. Infrastructure can be swapped behind a stable contract. Building pipelines for consumers that don't exist yet is premature (YAGNI) and adds cost with no decision behind it.

### 4c. Emit business events after the commit, not on submit

A business event should describe something that **actually happened**. If you emit it when the form is submitted, or before the database transaction commits, a failed validation or a rollback still produces an event, and the metrics overcount.

```md
<!-- telemetry-plan.md -->
- **System touchpoint:** Backend, inbound order creation. Emitted after the order and
  the stock increment are committed, never on form submit, so failed or rolled-back
  orders are not counted.
```

Failures get their **own** event instead (here `order_validation_failed`), so nothing is lost.

**Interview Q:** *The DB commit succeeds but the event send fails (or the reverse). How do you keep them consistent?* Use the **transactional outbox** pattern: write the event to an `outbox` table in the same transaction as the business data, and have a separate relay publish it afterwards. Combined with a unique `eventId`, consumers can deduplicate any retries (at-least-once delivery).

### 4d. Edge-triggered vs level-triggered alerts

- **Level-triggered**: fire on every check while the condition is true ("stock **is** below minimum").
- **Edge-triggered**: fire only when the condition **changes** from false to true ("stock **crosses** below minimum").

```md
<!-- telemetry-plan.md -->
- **Definition:** Fires when a product's stock at a clinic crosses from at or above its
  configured minimum to below it. It fires on the crossing only, not on every order while
  stock stays below the minimum.
- **System touchpoint:** ... (`previous_stock >= min_threshold` and `current_stock < min_threshold`).
```

Example: stock 3, minimum 10, five more single-item orders. Level-triggered = 5 alerts for one shortage; edge-triggered = 1.

**Interview Q:** *Why does this matter for alerting?* Level-triggered alerts on a persistent condition cause alert fatigue, so people start ignoring them. Edge-triggered gives one actionable alert per incident. It needs the previous state (before/after values), and you usually add a debounce for values flickering around the threshold.

---

## Event design

### 5. The Event Envelope

A fixed set of fields that **every** event carries, regardless of type. Event-specific data goes inside `properties`.

```md
<!-- plan.md -->
| Field           | Type              | Required | Notes                                   |
| --------------- | ----------------- | -------- | --------------------------------------- |
| `eventId`       | string (UUID)     | yes      | Idempotency and deduplication           |
| `timestamp`     | string (ISO 8601) | yes      | UTC recommended                         |
| `sessionId`     | string            | yes      | Browser or API session                  |
| `userId`        | string            | yes      | Authenticated operator; hash if needed  |
| `event_type`    | string            | yes      | `entity_action` taxonomy                |
| `schemaVersion` | string            | yes      | e.g. `1.0.0`                            |
| `requestId`     | string            | yes      | Correlates frontend, API, logs          |
| `properties`    | object            | yes      | Event-specific payload (allowlist only) |
```

**Interview Q:** *Why a common envelope?* The pipeline can route, deduplicate, validate, and join every event the same way, without knowing each event's type. Only `properties` varies.

### 6. Idempotency and deduplication with `eventId`

Networks retry, so the same event can arrive twice. A unique `eventId` (UUID generated at the source) lets the consumer drop duplicates.

**Interview Q:** *What does "at-least-once delivery" mean for telemetry?* Most pipelines guarantee at-least-once, not exactly-once, so duplicates will happen. Generate the ID on the client and deduplicate on `eventId` downstream to get effectively-once results.

### 7. Correlation with `requestId`

One ID passed through frontend → API → logs, so you can join everything that happened during one request.

**Interview Q:** *A user reports a slow order save. How do you trace it?* Find the `requestId` on the frontend event, then follow it to the API latency event and the server logs for that same request.

### 8. Schema versioning with `schemaVersion`

Events change over time. A version (semver, e.g. `1.0.0`) on each event lets consumers parse old and new payloads correctly.

**Interview Q:** *How do you change an event without breaking consumers?* Adding an optional field is a minor bump. Removing or renaming a field, or changing its type, is a major bump. Consumers branch on `schemaVersion`.

### 9. `entity_action` naming taxonomy

Event names follow `entity_action` in snake_case, past tense, with a consistent set of verbs.

```md
<!-- plan.md -->
inbound_order_created, stock_threshold_triggered, direct_stock_edit_rejected,
login_failed, session_expired, section_viewed
```

**Interview Q:** *Why does naming matter?* Without a convention you end up with `OrderCreated`, `order-create`, and `new_order` all meaning the same thing. Analysts can't find events, and dashboards silently miss data.

### 10. Property allowlist

An explicit list of keys allowed in `properties`. Anything not on the list is dropped or rejected. It's a security control, not just documentation.

```md
<!-- plan.md -->
| Property     | Type    | Required | Allowlist | PII |
| ------------ | ------- | -------- | --------- | --- |
| `clinic_id`  | string  | yes      | yes       | no  |
| `product_id` | string  | yes      | yes       | no  |
| `quantity`   | integer | yes      | yes       | no  |
```

**Interview Q:** *Allowlist vs blocklist for sensitive data?* A blocklist ("never send `patient_name`") fails open: any new field you didn't think of leaks. An allowlist fails closed: only known-safe fields get through.

---

## JSON Schema

### 11. JSON Schema draft-07 basics

A JSON document that describes the shape of other JSON: types, required fields, allowed values. Pipelines and SDKs use it to validate events.

**Interview Q:** *Where would you validate events against the schema?* At the producer in dev/CI (to catch bugs early) and at pipeline ingestion (to reject or quarantine bad events before they hit the warehouse).

### 12. `allOf` + `$ref`: reusing the envelope

Define the envelope once, then compose each event from it.

```md
<!-- plan.md -->
- one `eventEnvelope` definition with all 8 required fields and `"additionalProperties": false`
- one definition per event using `allOf` + `$ref` to the envelope, `event_type` as `const`,
  `properties` with `required` list and `"additionalProperties": false`
```

**Interview Q:** *What does `allOf` mean?* The data must be valid against **every** listed subschema. Envelope + event-specific rules = one event.

### 13. `const` and `additionalProperties: false`

- `const` fixes a field to one exact value (e.g. `"event_type": { "const": "login_failed" }`).
- `additionalProperties: false` rejects any key not declared, which is how the allowlist gets enforced in the schema.

**Watch the two meanings of `properties`:** in JSON Schema it's a keyword ("these are the allowed fields and their rules"); in our envelope it's also a field name (the event payload). Tell them apart by the parent: under a schema it's the keyword, listed next to `eventId` it's the field.

```json
// event-schemas.json
"eventEnvelope": {
  "properties": {                       // keyword
    "eventId":    { "type": "string", "format": "uuid" },
    "properties": { "type": "object" }  // envelope field (the payload)
  },
  "additionalProperties": false
}
```

**Interview Q:** *What's a gotcha with `additionalProperties: false` and `allOf`?* Each subschema only knows its own `properties`, so putting `additionalProperties: false` on the envelope can reject fields declared in the other branch. Apply it where the full set of keys is known (e.g. inside the `properties` object of each event).

### 14. Keeping the docs and the schema in sync

The Markdown plan and the JSON schema describe the same events, so they must match: names, properties, required flags.

**Interview Q:** *How do you stop docs and schema drifting apart?* Treat one as the source of truth and generate or check the other in CI. At minimum, review them side by side event by event before merging.

---

## Delivery strategy

### 15. Stream vs batch processing

- **Stream**: events are processed within seconds or minutes, because someone has to act now.
- **Batch**: events are processed periodically (hourly/daily), which is enough for reports and analytics.

```md
<!-- plan.md -->
| Urgency                     | Processing | Example                                      |
| --------------------------- | ---------- | -------------------------------------------- |
| Ops must act within minutes | stream     | stock threshold, repeated auth failures      |
| Daily/weekly reporting      | batch      | order volume aggregates, navigation heatmaps |
| Analytics only              | batch      | section popularity trends                    |
```

**Interview Q:** *Why not stream everything?* Streaming costs more to build, run, and keep on call. Choose it only when the business decision is time-critical. Base the choice on business urgency, not on what's technically nice.

### 16. Throttle vs debounce for high-frequency events

- **Debounce**: wait for things to go quiet, then emit once (e.g. one `stock_threshold_triggered` per product per clinic within 15 minutes).
- **Throttle**: emit at most once per time window, no matter how many triggers happen.
- **Sampling**: keep only a percentage (common for latency events).

**Interview Q:** *A stock level hovers around the threshold and fires 200 alerts an hour. What do you do?* Debounce per `(clinic_id, product_id)` with a time window, so ops gets one actionable alert instead of alert fatigue.

---

## Privacy and compliance

### 17. PII vs PHI, HIPAA and UK GDPR

- **PII**: personally identifiable information (name, email, IP).
- **PHI**: protected health information under HIPAA: anything linking a person to their health (names, record numbers, diagnoses).
- **UK GDPR** treats health data as a special category with stricter rules.

```md
<!-- plan.md -->
- Events describe **supplies and stock, never patients**.
- No `properties` field may contain patient names, medical record identifiers,
  diagnoses, or any real/simulated PHI.
- For clinical context, use only `department` or service type (e.g. `chronic_care`).
```

**Interview Q:** *How do you keep clinical context without PHI?* Aggregate to a non-identifying level: department or service type instead of patient. Consumption is tracked "by department, not by patient".

### 18. Data minimisation, anonymisation, and pseudonymisation

- **Minimisation**: capture only what a decision needs.
- **Anonymisation**: data can no longer be linked to a person at all.
- **Pseudonymisation**: replace the identifier with a token or hash (e.g. a hashed `userId`). It's still personal data under GDPR because it can be re-linked.

**Interview Q:** *Is hashing a user ID anonymisation?* No, it's pseudonymisation. A plain hash of a known ID can be reversed by hashing every candidate ID. Use a keyed hash (HMAC) with a secret salt, and still treat the result as personal data.

### 19. Risks and exclusions: deliberate scope cuts

A documented list of events you considered and dropped, plus data you will never capture (PHI, raw passwords, tokens), each with a reason.

**Interview Q:** *Why write down what you're NOT tracking?* It shows the scope was a choice, not an oversight. It stops someone from adding a risky field later, and it gives reviewers and auditors a clear record.

### 20. Closed enums instead of free text to block PHI

A field that carries context (like `department`) is a closed `enum`, not a free-text string. Free text lets someone type a patient's name into telemetry by accident. An enum makes that event impossible to emit. Adding a new value means a minor `schemaVersion` bump.

```json
// event-schemas.json — outbound_order_created
"department": { "type": "string", "enum": ["primary_care", "specialty_care", "chronic_care"] }
```

**Interview Q:** *How do you stop sensitive data leaking through a "harmless" context field?* Constrain it to a closed enum (plus `additionalProperties: false` for unknown keys), so the schema rejects anything outside the approved values instead of relying on people being careful.
