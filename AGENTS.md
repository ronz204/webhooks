# Webhook Delivery Lab: Agent Guide

A proof of concept that builds **both sides** of a webhook system: the **Emitter** (signed, retried, durable delivery) and the **Chaos Receiver** (a deliberately unreliable client). It succeeds when, after a large run through every kind of failure, the receiver's ledger equals the Producer's expected state. Delivery is at-least-once and unordered, yet the result is correct.

Stack: Bun, TypeScript, Elysia, Postgres + Drizzle, Redis + BullMQ, SQLite (receiver only), Docker Compose, `bun test`.

## Working rules

1. **Architecture:** DDD + Hexagonal. Layers `core` (domain), `app` (use cases, ports), `infra` (adapters). `core` and `app` never import `infra`.
2. **Process layout:** API and Worker share one codebase, run as separate processes from separate entrypoints.
3. **Terminology:** use the vocabulary in `context/foundations.md` exactly. Never invent synonyms for statuses, outcomes or components.
4. **Settled decisions stay settled.** Change anything in `context/agreements.md` only when the project owner asks.
5. **Assert invariants, not call sequences.** Tests check end states ("ledger equals expected").
6. **No hidden state.** Anything that must survive a crash lives in Postgres.
7. **Inject non-determinism.** Time, randomness and ids come through ports (`Clock`, `Random`, `IdGenerator`).
8. **Validate before relying.** Every new dependency gets a Bun compatibility check, recorded with the work that adds it.
9. **One phase, one branch:** `phase-<n>-<slug>`, complete only when its "done when" holds (phases and tasks in `ROADMAP.md`).
10. **Keep docs lean.** One fact has one home, link by ID. No folder trees, no code the project does not have yet.

## Hard invariants

- Postgres is the source of truth. Wiping Redis loses no delivery.
- Commit to Postgres first, enqueue second. Every enqueue uses the deterministic job id `<deliveryId>-<attemptNumber>`.
- Every attempt starts with the guarded claim. Zero rows claimed means "not mine": do nothing.
- One delivery per `(event, endpoint)`. The event id is identical in every attempt and replay.
- Never follow redirects. Never connect to an address that was not vetted.
- Sign and verify over raw bytes, compare in constant time. A secret is shown only when created or rotated.
- The lease is strictly longer than the delivery timeout.

Full list: INV-01 to INV-13 in `context/aggregates.md`.

## Avoid

- Working inside the webhook handler: persist and acknowledge, then process.
- Parsing before verifying the signature.
- Relying on delivery order.
- Letting the queue own business state: counts and statuses belong to Postgres.
- Treating every failure the same: `410`, `429` and `500` mean different things.
- Retrying forever or in lockstep: bound attempts, add jitter.
- Fixed sleeps in tests: poll for the outcome.
- Checking a hostname, then connecting by name: pin the address you checked.

## Where to read

Read only what the task needs. Cross-references use stable IDs with a 3-letter prefix: INV invariants and DOM domain rules (aggregates), DEC decisions (agreements), SCN scenarios (named in ROADMAP.md).

| If you are working on... | Read |
|---|---|
| What to build next, phase tasks and progress | `ROADMAP.md` |
| Orientation, vocabulary, goals, guarantees, risks, scope | `context/foundations.md` |
| Why something is built this way | `context/agreements.md` |
| Aggregates, invariants, statuses, retry policy, database guarantees | `context/aggregates.md` |
| Processes, layers, ports, queue rules, flows | `context/components.md` |

## Rules

Standing rules live in `.claude/rules/`. Claude Code loads them automatically. Any other agent should read the ones that match its task before writing code or committing. Frontmatter fields other than `paths` (tier, facets, enforce, upgrade, evidence, version) are metadata for a possible future rule-management tool, described in `artifacts/rules.artifact.md`. Claude Code ignores them.

| Rule | Applies when |
|---|---|
| `.claude/rules/code-lean-change.md` | You write or edit any source: how much code a change may add. |
| `.claude/rules/code-domain-step.md` | You work on the domain layer (any core or domain folder): contexts, entities, value objects, contracts. |
| `.claude/rules/code-english-only.md` | You name anything or write an error message or log line: always English. |
| `.claude/rules/commit-messages.md` | You make a commit: format, size, language, when to commit and how to group files. |

## Context file conventions

- English only, at most 150 lines per file, single-word names. No per-file metadata: this table is the index.
- `context/` is durable knowledge (why, rules, invariants) and stays. Write it as prose that explains, and avoid code formatting for ordinary words.
- Every ID has exactly 3 letters, a dash and a two-digit number (INV-01, DEC-21). Data ids follow the same rule (edp_, evt_, dlv_, att_, sec_).
- Each context document must be understandable on its own: assume no prior knowledge, define terms on first use and spell out acronyms.
- Decisions are Closed, Open or Superseded. Never delete a superseded one.
