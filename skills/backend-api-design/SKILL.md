---
name: backend-api-design
description: >-
  Design and implement production-grade backends and APIs — routes, service layers, validation, error handling, auth, pagination, and background jobs. Use this skill WHENEVER the user builds or modifies any server-side code: API endpoints, REST/RPC routes, Next.js API routes or server actions, Express/Fastify/NestJS services, Supabase edge functions, webhooks, or CRUD for any app. Trigger on phrases like "בקאנד", "צד שרת", "API", "endpoint", "ראוט", "שרת", "backend", "server", "database logic", or any feature that persists or fetches data — even if the user only says "add a feature that saves X".
---

# Backend & API Design

Purpose: every backend Claude writes should be safe to expose to the internet on day one. This skill defines the non-negotiable structure and the decision points.

## Core Architecture Rule: Three Layers, Always

Never put business logic inside a route handler. Even in a "small" project, structure as:

```
route/handler   → parse input, call service, shape response. NO logic.
service         → business rules, orchestration, transactions.
data access     → queries only. No business decisions.
```

Why: routes change with frameworks, services don't. Testing a service is trivial; testing logic buried in a handler requires spinning up HTTP. When the user later says "add a CLI/cron/webhook that does the same thing," the service is reused as-is.

Small-project exception: a single-file prototype under ~200 lines may inline layers, but state this tradeoff to the user explicitly.

## Input Validation: At the Boundary, Never Inside

Every external input (body, query, params, headers, webhook payloads) is validated at the edge with a schema library (zod for TS, pydantic for Python). After the boundary, code operates on typed, trusted data.

```ts
const CreateOrder = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().min(1).max(100),
});
// handler:
const input = CreateOrder.safeParse(await req.json());
if (!input.success) return json({ error: "VALIDATION", details: input.error.flatten() }, 400);
```

Rules:
- Whitelist fields (`.strict()` / explicit schema). Never spread `req.body` into a DB write — that's mass assignment.
- Coerce and bound numbers. Unbounded `limit` params are a DoS vector.
- Validate webhook signatures (Stripe, GitHub, etc.) BEFORE parsing the payload.

## Error Handling Contract

Define one error shape for the whole API and never leak internals:

```json
{ "error": { "code": "ORDER_NOT_FOUND", "message": "Human-readable, safe to show", "requestId": "..." } }
```

- Machine-readable `code` (SCREAMING_SNAKE), stable across versions. Clients switch on codes, never on message text.
- 4xx = caller's fault, 5xx = ours. Never return 200 with `{ok:false}`.
- Log the full error server-side with a requestId; return only the requestId to the client. Stack traces in responses are an information-disclosure bug.
- Distinguish expected failures (return typed error) from bugs (throw, let global handler convert to 500).

## Auth & Authorization

- Authentication: verify identity once, in middleware. Attach `user` to context.
- Authorization: check permissions in the SERVICE layer per resource, not just per route. "Is this user allowed to touch THIS order" (object-level, IDOR prevention) is the check most AI-generated code misses. Every query that fetches a user-owned resource includes the owner filter:

```ts
// Correct — ownership enforced in the query itself
db.order.findFirst({ where: { id, userId: ctx.user.id } });
// Wrong — fetch then check, or worse, no check
```

- Never trust client-supplied role/userId fields. Derive identity from the session/token only.
- Read the companion `app-security` skill for the full checklist when handling auth flows, passwords, or tokens.

## API Shape Conventions

- Resources are plural nouns: `GET /orders/:id`, `POST /orders`. Actions that don't map to CRUD get a verb sub-resource: `POST /orders/:id/cancel`.
- Pagination is mandatory on every list endpoint from day one. Prefer cursor-based for feeds, offset for admin tables. Response always includes `nextCursor`/`total`.
- Timestamps: ISO 8601 UTC in transport, always. Convert at the UI.
- IDs: UUIDs or prefixed IDs (`ord_x7Kq...`) in the API even if the DB uses serials — sequential IDs leak volume and enable enumeration.
- Versioning: don't add `/v1` ceremonially to a solo project; DO keep responses additive (add fields, never repurpose or remove without a version bump).

## Idempotency & Side Effects

- Any endpoint that charges money, sends email, or creates external resources accepts an `Idempotency-Key` header and stores results keyed on it. Retries happen — networks fail after the server succeeds.
- Webhook handlers must be idempotent by design (upsert on external event ID) because providers redeliver.
- Slow work (email, PDF generation, media processing) goes to a queue or background function; the request returns 202 with a job ID. A request handler that takes >5s is a design smell.

## Data Mutations

- Multi-step writes go in a transaction. If step 2 can fail, step 1 must roll back.
- Prefer soft-state transitions over deletes for anything with money/audit implications (`status: 'cancelled'` not `DELETE`).
- Race conditions: for check-then-write patterns (unique username, inventory decrement), rely on DB constraints/atomic updates, not application-level checks:

```sql
UPDATE inventory SET count = count - 1 WHERE product_id = $1 AND count > 0; -- atomic
```

## Response Discipline

- Never return the raw DB row. Map to an explicit response type — this is where password hashes, internal flags, and other-users' data leak.
- Define the response shape as a type/schema next to the route. The mapping function is the single place fields are exposed.

## Checklist Before Declaring a Backend Feature Done

Run through this explicitly and report status to the user:

1. All inputs schema-validated at the boundary, fields whitelisted
2. Object-level authorization on every resource access (IDOR check)
3. Errors follow the contract; no stack traces or internals in responses
4. List endpoints paginated and bounded
5. Money/email/external side effects are idempotent
6. Multi-step writes are transactional
7. Responses mapped through explicit types, no raw rows
8. Secrets from env only (see ship-and-deploy skill), never in code
9. At least one happy-path and one auth-failure test exist (see testing-strategy skill)

## Framework Notes

- **Next.js**: server actions are API endpoints — validate and authorize them like any route; "it's only called from my UI" is false, anyone can invoke them directly.
- **Supabase**: RLS is your authorization layer — enable it on every table, write policies before writing queries, and never ship with the service-role key in client-reachable code.
- **Express/Fastify**: centralize the error handler and the auth middleware; per-route try/catch sprawl is a smell.
