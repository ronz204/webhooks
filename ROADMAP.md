# Roadmap

The path from an empty repository to a laboratory that proves its own correctness. Work happens in six phases, in order.

**How to use this file**
- Each phase ships on its own branch, named phase-n-slug, and is complete only when its **Done when** condition holds.
- Tick a task when it lands. Update the phase status in the same commit.
- Scenario IDs (SCN-01 to SCN-18) refer to the end-to-end scenarios that prove each phase.

**Status:** ⬜ not started · 🟨 in progress · ✅ done

---

## Phase 0: Foundations ⬜

**Goal:** prove the plan is buildable on Bun before writing any domain code.

- [ ] Docker Compose with Postgres, and Redis configured with append-only persistence and the noeviction memory policy
- [ ] Project skeleton with the core, app and infra layers and separate API and Worker entrypoints
- [ ] Typed configuration, loaded once and validated at startup, including the rule that the lease exceeds the delivery timeout
- [ ] Lint rule that stops core and app from importing infra
- [ ] Bun compatibility check for each dependency: BullMQ with its Redis client, the Postgres driver with Drizzle, Elysia, the ULID generator, SQLite, HMAC signing, DNS resolution
- [ ] Spike for HTTPS pinned to a pre-resolved IP: try Bun fetch, fall back to the Node https module with a custom lookup, and record the choice (DEC-18)
- [ ] Test harness: fast configuration profile, an "eventually" polling helper, and a distinct queue prefix per scenario

**Done when:** every check passes or has a recorded fallback, and the whole stack starts with one command.

## Phase 1: Publish and fan-out ⬜

**Goal:** accept endpoints and events, and create deliveries transactionally.

- [ ] Domain model: Endpoint, Event and Delivery aggregates with their invariants and allowed transitions, covered by unit tests
- [ ] Postgres schema and migrations with the constraints that back the invariants
- [ ] Endpoint registration with secret generation, plus listing, reading and updating
- [ ] Event publishing with producer idempotency keys, including the conflict case and concurrent identical publishes
- [ ] Transactional fan-out: one delivery per matching endpoint, held for disabled endpoints
- [ ] Cursor-based event listing with the settle window
- [ ] API key authentication, the standard error format and the health check

**Done when:** publishing an event creates exactly one delivery per matching endpoint, republishing with the same key returns the original event, and SCN-10 passes.

## Phase 2: Happy-path delivery ⬜

**Goal:** deliver successfully, end to end.

- [ ] BullMQ queue adapter with deterministic job ids, and enqueueing after commit
- [ ] Worker process with the guarded claim and the persisted attempt row
- [ ] Request construction and HMAC signing
- [ ] HTTP dispatcher with timeout, no redirects and a bounded response
- [ ] Outcome classification and the move to delivered
- [ ] Delivery listing and detail, and the stats endpoint
- [ ] Chaos Receiver with the healthy behavior: signature verification, inbox, applier loop and ledger
- [ ] Producer commands: produce and verify

**Done when:** SCN-01 passes and verify reports a matching ledger.

## Phase 3: Failures and recovery ⬜

**Goal:** survive failure.

- [ ] Retry policy with exponential backoff, jitter and Retry-After
- [ ] Dead status when attempts are exhausted
- [ ] Leases and the Sweeper: re-enqueue overdue pending deliveries and recover expired leases
- [ ] Graceful Worker shutdown
- [ ] Receiver behaviors: flaky, lost-ack, slow, rate-limited and drop
- [ ] Receiver control interface: change behaviors, switch the outage, reset, read stats
- [ ] Safety checks for several Workers running at once

**Done when:** SCN-02, SCN-03, SCN-04, SCN-07, SCN-08, SCN-09, SCN-17 and SCN-18 pass.

## Phase 4: Robust receiver and convergence ⬜

**Goal:** prove exactly-once effect and finish the recovery paths.

- [ ] Receiver deduplication on the event id and version-based profile updates that tolerate disorder
- [ ] Endpoint-scoped event listing on the Emitter
- [ ] Receiver reconciliation by cursor
- [ ] Manual delivery replay, with attempt numbering that continues
- [ ] Producer verify that can trigger reconciliation
- [ ] The 1,000-event convergence run

**Done when:** SCN-05, SCN-06, SCN-11 and SCN-12 pass, and the 1,000-event run matches the expected ledger.

## Phase 5: Security, endpoint health and operations ⬜

**Goal:** harden the Emitter.

- [ ] Target guard that blocks private addresses, with IP pinning
- [ ] Verified redirect refusal
- [ ] Secret rotation with an overlap window
- [ ] Endpoint disable and enable, with held backlogs
- [ ] Handling of 410 Gone
- [ ] Automatic disabling after repeated failures
- [ ] Development-only queue dashboard

**Done when:** SCN-13, SCN-14, SCN-15 and SCN-16 pass.

---

## Project complete

- [ ] All 18 scenarios pass repeatedly without manual intervention
- [ ] The 1,000-event run converges
- [ ] Wiping Redis or killing a Worker mid-attempt loses no delivery
- [ ] Private and loopback endpoints are rejected when private targets are not allowed
