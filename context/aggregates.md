# Aggregates

The domain model of the Emitter (the webhook delivery service this project builds): what exists, which rules it must always follow, how a delivery moves through its life, and what the database has to guarantee. It deliberately says nothing about Redis, the HTTP framework or any other technology, so it stays true if those change.

## Key terms

- **Aggregate:** a cluster of data treated as one unit, with its own identity and its own rules. Other parts of the system change it only through its behavior.
- **Invariant:** a rule that must be true at every moment. Tests and code reviews check against them.
- **Idempotent:** safe to repeat. Doing it twice has the same effect as doing it once.
- **Guarded claim:** a single conditional database update that lets a worker start an attempt only if the delivery is in the expected state. If it matches nothing, the worker does nothing.
- **Lease:** a deadline written on a delivery when an attempt starts. If the worker dies, the deadline passes and the delivery can be recovered.
- **Sweeper:** a background task that finds overdue or abandoned work and puts it back in motion.
- **Backoff and jitter:** waiting longer after each failure, with a small random variation so many failures do not retry at the same instant.

## The three aggregates

Each aggregate protects its own rules, and they refer to each other by id only. Ids are prefixed ULIDs (unique ids that sort by creation time): edp_ for endpoints, evt_ for events, dlv_ for deliveries, att_ for attempts and sec_ for secrets.

**Endpoint** is a subscription. It holds the destination URL, the event types it wants (exact types, or a star meaning everything), a description, a status of active or disabled, the reason it was disabled (manual, gone, or too_many_failures), a counter of consecutive failures, and its signing secrets. It owns at most two non-expired secrets at once, each with a value, a creation time and an optional expiry.

**Event** is an immutable fact. It holds a type, a JSON payload, a creation time and an optional idempotency key supplied by the producer. It also has a sequence number that only grows, used as a pagination cursor. Once created it never changes, and its id is what receivers use to recognize duplicates.

**Delivery** is the obligation to deliver one event to one endpoint, unique per pair. It holds its status, the number of attempts made, the attempt limit, the time of the next attempt, the lease expiry, the last outcome with its HTTP status, and the delivered time. It owns its **Attempts**: each has a number, start and finish times, an outcome, an HTTP status, a duration and a short response snippet.

## Invariants

| # | Rule | Why it matters |
|---|---|---|
| INV-01 | At most one delivery per event and endpoint pair. | We never send the same event twice to a subscriber by our own mistake. |
| INV-02 | An event never changes, and its id is identical in every attempt and replay. | Receivers recognize duplicates by this id. |
| INV-03 | A delivery changes status only along the transitions in DOM-02. | The state machine stays predictable and testable. |
| INV-04 | At most one attempt is in progress per delivery, enforced by the guarded claim and a unique constraint on delivery and attempt number. | Two workers never run the same attempt at once. |
| INV-05 | Attempt numbers start at 1 and grow by exactly 1, across replays too. | The history has no gaps. |
| INV-06 | Postgres is the source of truth. | Wiping Redis loses no delivery. |
| INV-07 | Every pending or in_flight delivery is eventually executed or recovered, even if its queue job is lost. | Nothing stays stuck forever. |
| INV-08 | Redirects are never followed, and no connection is made to a non-public address unless private targets are explicitly allowed. | Blocks server-side request forgery. |
| INV-09 | A secret value is returned only by the response that creates or rotates it. | Limits the chance of leaking it. |
| INV-10 | The lease duration is strictly greater than the delivery timeout. | A slow but healthy attempt is not mistaken for a dead one. |
| INV-11 | The receiver applies each event id at most once. | Duplicate deliveries cause no double effect. |
| INV-12 | After delivery quiesces and the receiver reconciles, its ledger equals the state the Producer expected. | This is the project's final success test. |
| INV-13 | A disabled endpoint receives no new attempts. An attempt already in flight may finish. | We stop hammering a broken endpoint without cutting a request in half. |

## DOM-01 Matching and fan-out

An event matches an endpoint when the endpoint wants everything or lists the event's type. When an event is published, one transaction stores the event and creates a delivery for every matching endpoint. Matching active endpoints get a pending delivery due immediately, with no attempts made and an attempt limit equal to the number of retry delays plus one. Matching disabled endpoints get a held delivery instead.

## DOM-02 Delivery lifecycle

Only the transitions below are legal. The aggregate rejects every other one.

| From | To | When it happens |
|---|---|---|
| (new) | pending | Fan-out for an active endpoint. |
| (new) | held | Fan-out for a disabled endpoint. |
| pending | in_flight | A worker wins the guarded claim. |
| in_flight | delivered | The attempt succeeded. |
| in_flight | pending | The attempt failed in a retryable way and attempts remain. |
| in_flight | dead | The attempt failed in a retryable way and no attempts remain. |
| in_flight | held | The endpoint answered 410, or a retryable failure hit an endpoint that is disabled, including one disabled by this very failure. |
| pending | held | The endpoint was disabled. |
| held | pending | The endpoint was enabled. |
| dead or delivered | pending | A manual replay. |

When the Sweeper recovers an expired lease, it treats it as a retryable failure with the outcome lease_expired, so the delivery follows the same rows: pending, dead or held.

An endpoint moves from active to disabled for one of three reasons: someone disabled it, it answered 410, or it failed too many times in a row. Enabling it returns it to active, clears the reason and resets the failure counter.

## DOM-03 Retry policy

After a retryable failure, let n be the number of attempts made so far.

1. If n has reached the attempt limit, the delivery becomes dead.
2. Otherwise take the delay at position k in the schedule, where k counts the attempts used in the current budget (a replay starts a fresh budget).
3. Apply jitter: multiply the delay by one plus a random fraction between minus and plus the configured jitter.
4. If the outcome was rate_limited and the response carried a valid Retry-After (seconds or an HTTP date, anything else ignored), wait at least that long, but never more than the largest delay in the schedule.
5. Set the next attempt time, return the delivery to pending, and after the commit enqueue a delayed job for attempt n plus one.

The default schedule is 10 seconds, 1 minute, 5 minutes, 30 minutes, 2 hours, 6 hours and 24 hours: seven waits, so eight attempts spanning about 32 hours. A replay sets the attempt limit to the current attempt count plus a full budget, so attempt numbering keeps growing while the budget starts fresh.

## DOM-04 Outcome classification

Not every failure is the same, and the response decides what happens next.

| Observation | Outcome | What happens | Counts against endpoint health |
|---|---|---|---|
| Any 2xx | success | Delivered. | No, and it resets the counter. |
| 410 | gone | Held, and the endpoint is disabled with reason gone. | Not applicable. |
| 429 | rate_limited | Retry, honoring Retry-After. | No, the endpoint is alive. |
| 3xx | redirect | Retry. The redirect is never followed. | Yes. |
| Any other status | http_error | Retry. | Yes. |
| No response in time | timeout | Retry. | Yes. |
| Connection, DNS or TLS failure | network_error | Retry. | Yes. |
| Guard refused the address | blocked_target | Retry later, since DNS may change. | Yes. |
| Lease expired | lease_expired | Retry. Set by the Sweeper, not by a response. | No. |
| Unexpected processor exception | internal_error | Stays in_flight until the lease expires and the Sweeper recovers it. | No. |

The last two are the Emitter's own problems, so they never count against the endpoint.

## DOM-05 Secrets

A secret is the text whsec_ followed by the base64 of 32 random bytes, and the signing key is those decoded bytes. A new endpoint starts with one secret. Rotation adds a second one and gives the previous one an expiry, so at most two non-expired secrets exist at any time and receivers can switch without downtime. The value is shown only when an endpoint is created or a secret is rotated, and never appears in any other response, log or error message.

## What the database must guarantee

Column lists belong in the schema once it exists. These rules are what any schema has to enforce.

- Deliveries are unique per event and endpoint pair (INV-01), and attempts are unique per delivery and attempt number (INV-04, INV-05).
- Events carry an automatically growing sequence number, which is the cursor for listing and reconciliation.
- An idempotency key is unique when present, and the event stores a hash of its type and payload, so reusing a key with different content is detected.
- Partial indexes on the next attempt time of pending deliveries and on the lease expiry of in-flight deliveries let the Sweeper find overdue work quickly. An index on endpoint and status serves holding and releasing.

Some operations must happen in a single transaction: publishing (the event and all its deliveries), claiming (the claim and the new in-progress attempt), finishing an attempt (the attempt, the delivery transition and the endpoint health, per DEC-14), recovering a lease (selection that skips rows locked by other Sweepers, the expired attempt and the transition), and disabling or enabling an endpoint (its status and the move of its deliveries between pending and held). Queue work always happens after the commit.
