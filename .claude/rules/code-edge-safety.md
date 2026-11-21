---
paths:
  - "**/infra/**"
  - "**/adapters/**"
  - "**/clients/**"
  - "**/config/**"
tier: architecture
facets: [hexagonal]
enforce: text
upgrade: test
evidence: unmeasured
version: 1
---

# Edge Safety Conventions

This governs code that crosses a boundary: anything that talks to the network, a database, a queue, a file, a person, or the process configuration. It states what must hold at every boundary whatever the technology. How to meet each point with a particular technology belongs to a technology playbook, when the project has one.

---

## At every boundary

- Validate on the way in, then trust. Anything that comes from outside (a response, a row, a message, an environment variable) is checked once, at the edge, against an explicit schema, so inner code never sees an unvalidated shape. Anything that claims an identity (a signature, a token) is authenticated over the raw input before it is parsed. Configuration is validated at startup, and the process refuses to start when it is inconsistent.
- Bound time and size. Every outbound call has a timeout and every read has a maximum size, because an unbounded wait or buffer lets one slow or hostile party exhaust the process.
- Make repeats safe. Whatever can be delivered, retried or re-run twice (a job, a request, a message) has the same effect the second time. Give the operation a deterministic identity and let the store reject the duplicate, because an in-memory check is lost with the process.
- Translate failures. An adapter turns every failure of its technology into the domain error its contract promises, with a stable code, so callers decide what to do without knowing what failed. An expected failure (timeout, not found, rate limit) is a result to classify, not an unexpected exception.
- Keep survivable state in the system of record. State that must outlive a crash lives in the durable store. A cache, a queue or an in-memory copy may speed things up, but it must be rebuildable from the system of record.
- Never log or return secrets. Logs are structured and messages are fixed per code. Secrets, credentials, signatures and full payloads never appear in a log line, an error message or a response, except in the one place that deliberately issues them.

---

## Non-goals

- Does not choose technologies or show how to implement a point with one.
- Does not govern the domain layer's structure or any business rule.
