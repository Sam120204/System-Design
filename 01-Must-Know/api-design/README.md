# API Design

An API is the contract between a caller and a service. For each operation, specify what the caller sends, what the service returns, who has access, and what happens if the caller repeats the request.

## Choose the API style

![API choice flowchart using client-facing versus internal calls and the need for flexible response fields](https://www.hellointerview.com/_next/static/media/CFM9z3MLfwiS.0udf3y5ph31tw.svg)

| Style | Good fit | What to plan for |
| --- | --- | --- |
| REST over HTTP | A web or mobile client reads and modifies identifiable resources | Resource paths, HTTP methods, status codes, and cache policy |
| GraphQL | Clients need different views of the same related objects | Schema and resolver ownership, authorization, query limits, and batching |
| RPC, such as gRPC | Services call well-defined operations using a shared contract | Schema compatibility, deadlines, and streaming support |

An event page could fetch an event and its venue over REST. A GraphQL client can request the event name and selected venue fields together. For live updates, choose SSE for server-to-client events or WebSockets when both sides send frequent messages. [GraphQL](../networking-fundamentals/08-graphql.md) · [SSE](../networking-fundamentals/10-server-sent-events.md) · [WebSockets](../networking-fundamentals/11-websockets.md)

### Ticketmaster example

Use **REST APIs** for public endpoints consumed by mobile apps and web clients. Use **gRPC** for internal communication between the booking, payment, and inventory services. For example, the client calls `POST /events/123/bookings` over HTTP. The booking service can then call the inventory service's `ReserveSeats` and the payment service's `AuthorizePayment` over gRPC. [REST](../networking-fundamentals/07-rest-apis.md) · [gRPC](../networking-fundamentals/09-grpc.md)

## Model resources and inputs

| Operation | Request | Expected result |
| --- | --- | --- |
| List upcoming events | `GET /events?city=Toronto&status=upcoming` | `200 OK` with a page of events |
| Read seat inventory | `GET /events/123/tickets` | `200 OK` with the available view |
| Create a booking | `POST /events/123/bookings` | `201 Created` with its ID and a `Location` header |
| Read a booking | `GET /bookings/bk_42` | `200 OK` for an authorized caller |

Use the **path** for resource identity and required scope, **query parameters** for filters, sorting, and pagination, **headers** for metadata such as authorization and idempotency, and the **body** for structured input. With `POST /events/123/bookings`, the event ID is part of the path, the body contains requested seats, and the service obtains the customer identity from validated credentials. Return a purpose-built response so internal schema changes or fields can stay private.

`GET` has safe read semantics. `PUT` replaces the representation at a known URI, `PATCH` applies a partial change, and `DELETE` removes the resource's association with its URI. `PUT` and `DELETE` have idempotent semantics even when repeated calls return different status codes. The retry behavior of a `PATCH` depends on the change it applies. [HTTP semantics](https://www.rfc-editor.org/rfc/rfc9110.html)

## Design lists, failures, and updates

- **Pagination.** Offset pagination is simple and supports page numbers. Large offsets can require skipping many rows, and concurrent changes can shift results. For a large or changing event feed, use an opaque cursor over a stable order such as `(starts_at, event_id)` and a bounded `limit`. Keep filters consistent across pages and define snapshot behavior when clients need a fixed view.
- **Errors.** Use HTTP status codes for the failure category and a stable application code for client handling. A booking conflict could return `409 Conflict` and `{"error":{"code":"SEAT_UNAVAILABLE","message":"Select another seat."}}`. Keep internal diagnostics out of public responses.
- **Compatibility.** Prefer additive changes where existing clients tolerate them. Preserve field meanings, set predictable defaults, and give clients a path to migrate before removing or changing a contract they use. Versions may be expressed in paths, headers, or media types.

### Idempotency Keys

A booking request can time out after the service has already reserved the seat or authorized payment. The client generates an `Idempotency-Key` for each logical booking and sends the same key and request inputs when retrying that booking.

```http
POST /events/123/bookings
Content-Type: application/json
Idempotency-Key: 8e03978e-740c-4c2e-abd1-52f427d5c177

{"seat_ids":["A-12"]}
```

1. **Claim the key.** Atomically claim it in durable storage, scoped to the caller and operation, before performing side effects. Bind it to the request inputs.
2. **Coordinate overlapping attempts.** If the same request arrives while the booking is in progress, wait for the original or return an in-progress response. Reject a key reused with different inputs.
3. **Reuse the outcome.** Once the booking completes, a retry can return the original booking ID and result. Use a stable operation ID for the internal inventory and payment calls so they can deduplicate their own retries and recover an uncertain outcome after a crash.

Document how long the service retains the key and result. The retention period should cover the supported retry window. [Timeouts and retries](../networking-fundamentals/15-timeouts-retries-backoff-and-jitter.md) · [Payment design case](../../07-Internal-Consistency-System/design-payment-stripe.md)

## Secure and protect the contract

**Authentication** establishes caller identity. API keys commonly identify an application or integration, and signed access tokens can represent a user or service. A JWT-based design verifies the signature and relevant claims, including issuer, audience, and expiration. **Authorization** checks the operation and resource after authentication. A customer may view their own booking, while a venue manager can be limited to bookings for their venues. Roles and ownership checks serve different parts of that policy.

Rate-limit by the identity and operation that consume the constrained resource. A token bucket can permit short bursts while controlling the sustained rate. Return `429 Too Many Requests` when a caller exceeds its limit, with `Retry-After` when the service can suggest a wait.

## Source

[API Design · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/api-design)
