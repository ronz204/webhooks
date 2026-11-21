# Agreements

This document records every design decision the project owner has settled, and the reason behind each one. Read it when you wonder "why is it built like this?", or before proposing a change. You do not need to have read the other documents: terms are explained where they first appear.

**Statuses.** **Closed** means settled: it changes only when the project owner asks, and the change is recorded here. **Open** means not decided yet. **Superseded by DEC-nn** means replaced by a later decision, and it is never deleted so the history stays readable. All 21 decisions below are Closed and were settled on 2026-11-20.

**Shape.** Every decision has a **Decision** (what we do) and a **Why** (the reason). Some also have a **Rejected** line (the alternative we did not choose).

## Quick index

| ID | Topic | In short |
|---|---|---|
| DEC-01 | Source of truth | Postgres is the truth, Redis only speeds things up |
| DEC-02 | Queue | BullMQ on Redis |
| DEC-03 | Retry ownership | One job per attempt, the domain decides the delay |
| DEC-04 | Two systems | Commit to Postgres first, enqueue second |
| DEC-05 | Delivery semantics | At-least-once plus idempotency |
| DEC-06 | Ordering | Not guaranteed |
| DEC-07 | Signing | HMAC-SHA256 with Standard Webhooks headers |
| DEC-08 | Receiver isolation | Separate process, database and code |
| DEC-09 | Disabled endpoints | Deliveries are held until re-enabled |
| DEC-10 | Time in tests | Real timers, compressed schedule |
| DEC-11 | Missed events | Pull reconciliation for one endpoint |
| DEC-12 | Secrets storage | Plain text |
| DEC-13 | Authentication | One static API key |
| DEC-14 | Attempt-finish transaction | One transaction updates two aggregates |
| DEC-15 | Event cursor | A settle window hides very recent events |
| DEC-16 | Process layout | One codebase, two entrypoints |
| DEC-17 | Scenario independence | Each scenario starts from scratch |
| DEC-18 | IP pinning | Bun fetch first, Node https as fallback |
| DEC-19 | Slow endpoints | Short timeout, high concurrency |
| DEC-20 | Documentation | English, short, self-contained files |
| DEC-21 | Stack | Bun, TypeScript, Elysia, Postgres, Redis, SQLite |

## Storing data and running work

### DEC-01 Source of truth
**Decision:** Postgres (the database) holds endpoints, events, deliveries and attempts. Redis (the fast in-memory store behind the queue) only speeds up execution and never holds anything Postgres does not.
**Why:** Redis can lose data depending on its persistence settings. Postgres gives transactions, full history, and the ability to ask questions such as "what failed today?".

### DEC-02 Queue
**Decision:** The queue is BullMQ, a job-queue library built on Redis.
**Why:** It provides delayed jobs (run later), concurrency control and per-job visibility, so we do not build them ourselves.

### DEC-03 Retry ownership
**Decision:** Each attempt is its own queue job and runs exactly once, with no automatic queue-level retries. The domain code computes the delay before the next attempt, and the Worker then enqueues a new delayed job.
**Why:** Retry state, attempt count and backoff (waiting longer after each failure) stay in one place instead of being split between BullMQ counters and the database. Honoring a Retry-After header, the receiver's own "wait this long" hint, also becomes straightforward.
**Rejected:** letting BullMQ retry failing jobs by itself.

### DEC-04 Writing to two systems
**Decision:** Commit to Postgres first and enqueue second. If the enqueue fails or the process dies in between, the Sweeper (a background task that repairs inconsistencies) finds the delivery later and enqueues it. Job ids are deterministic, so enqueueing the same attempt twice is harmless.
**Why:** One transaction cannot span two different systems, a gap known as the dual-write problem.
**Rejected:** enqueueing before the commit, because it creates ghost jobs that point to rows which do not exist.

### DEC-12 Secrets storage
**Decision:** Signing secrets are stored as plain text in the database.
**Why:** This is a laboratory, so encryption at rest is out of scope.

### DEC-14 The attempt-finish transaction
**Decision:** When an attempt ends, one transaction updates both the Delivery and the Endpoint's failure counter, even though they are separate aggregates (units of data with their own rules). It is the only place that crosses aggregates.
**Why:** The alternative, eventual consistency through domain events, adds machinery a laboratory does not need.

## What the system promises

### DEC-05 Delivery semantics
**Decision:** The promise is at-least-once delivery (every message arrives, possibly more than once) plus idempotency on the receiving side (applying a message twice has the same effect as once).
**Why:** Exactly-once delivery is not achievable across an unreliable network, so the project aims for exactly-once effect instead.

### DEC-06 Ordering
**Decision:** Delivery order is not guaranteed. Receivers tolerate disorder with commutative effects (operations whose order does not change the result, such as adding amounts) and versioned state.
**Why:** Guaranteeing order would force one-at-a-time delivery per endpoint, where one stuck message blocks all the others behind it.

### DEC-09 Disabled endpoints
**Decision:** Deliveries for a disabled endpoint become held. They are never discarded or failed, and they stay held until someone enables the endpoint, with no automatic expiry.
**Why:** Enabling the endpoint then delivers the whole backlog. Cleaning up old data is out of scope.

### DEC-11 Missed events
**Decision:** The safety net is reconciliation: the receiver pulls the events it missed from the Emitter, page by page, using a cursor (a bookmark). It reconciles one endpoint only, identified by a single configured endpoint id.
**Why:** Webhooks alone can be lost, for example when a receiver is down longer than the retry window. A single endpoint keeps the receiver simple.

### DEC-15 The event cursor
**Decision:** Event listings hide events younger than a short settle window.
**Why:** Sequence numbers are assigned when a row is inserted but become visible when its transaction commits, so a lower number can appear after a higher one. Without the window, a reader that already moved past the higher number would skip the lower one forever.

## Security and access

### DEC-07 Signing
**Decision:** Requests are signed with HMAC-SHA256 (a signature computed from a shared secret) over the event id, the timestamp and the body, sent in three webhook-prefixed headers, following the Standard Webhooks convention (an open specification for webhook signing).
**Why:** It is interoperable and well understood, and one header can carry several signatures, which is what makes secret rotation possible.

### DEC-13 Authentication
**Decision:** One static API key protects the Emitter.
**Why:** It is enough for a single-tenant laboratory.

### DEC-18 IP pinning
**Decision:** To defend against SSRF (server-side request forgery, tricking our server into calling internal addresses), the Worker connects to the exact IP address it vetted, while still sending the original hostname for the Host header and TLS. The Phase 0 spike tries Bun's fetch first, and the fallback is the Node https module with a custom lookup function.
**Why:** Checking an address and then connecting by name lets DNS change the answer in between (DNS rebinding). The decision is Closed because the fallback is already chosen. The spike only picks which one ships.

## Structure and process

### DEC-08 Receiver isolation
**Decision:** The Chaos Receiver is a separate process with its own SQLite database, and it shares no code with the Emitter.
**Why:** It has to behave like a real third-party client.

### DEC-16 Process layout
**Decision:** The API and the Worker share one codebase but start from separate entrypoints.
**Why:** They can then scale independently.

### DEC-19 Slow endpoints
**Decision:** The defense is a short delivery timeout and worker concurrency sized above the number of simultaneously slow requests. There is no per-endpoint concurrency control.
**Why:** A slow endpoint could occupy worker slots and delay healthy ones, but a full solution is more than a laboratory needs. Scenario SCN-04 measures the effect.

### DEC-21 Stack
**Decision:** Bun and TypeScript, Elysia for HTTP, Postgres through Drizzle (a database toolkit), Redis with BullMQ, SQLite for the receiver, Docker Compose, and the Bun test runner.
**Why:** The project owner chose it. Phase 0 validates that every piece works on Bun.

## Testing and documentation

### DEC-10 Time in tests
**Decision:** Scenario tests use real timers with a compressed retry schedule, plus a Clock port (an interface for asking the time) so pure domain tests can control time.
**Why:** BullMQ delays run on real time, so shrinking the schedule is more honest than faking the clock.

### DEC-17 Scenario independence
**Decision:** Each scenario starts from empty tables, a reset receiver and its own queue key prefix.
**Why:** The suite stays reliable in any order.

### DEC-20 Documentation
**Decision:** Everything is written in English, in files of at most 150 lines, with every fact living in exactly one place. Each context document must be understandable on its own, by someone who has not read the others: define terms on first use and spell out acronyms. Every ID has a 3-letter prefix. The context folder holds durable knowledge. The sandbox folder holds specifications for code that does not exist yet, and each is shrunk or deleted once the code, schema or tests replace it.
**Why:** Research on agent context files shows that only what an agent cannot discover from the code is worth writing, and that anyone, human or agent, may open any single file first.
