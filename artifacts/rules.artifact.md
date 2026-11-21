# Rule Packs: idea for a separate project

> **Status:** parked idea, not started. Written down so it is not lost. Nothing in the webhooks project depends on it.

## The problem

Rules that constrain a coding agent depend on the kind of project. Variability comes from several independent dimensions:

- **Stack:** queues, databases, HTTP frameworks, test runners.
- **Architecture style:** DDD, hexagonal, plain MVC, scripts.
- **Domain:** payments, webhooks, content sites.
- **Maturity:** a new project, or a legacy one.
- **Team and personal conventions:** commit style, language of identifiers.

Writing one rule file per combination explodes quickly. Copying rules between repositories makes them drift. Loading every rule in every session costs tokens and lowers adherence. Rules can also contradict each other, and nobody notices until the agent behaves oddly.

## Core idea

A **library of small, composable rule packs**, plus a **manifest** that says what a project is, a **detector** that proposes the manifest, and a **resolver** that compiles only the matching packs into the format each agent reads.

1. **Packs.** Each pack covers one topic, written as imperative rules with the reason attached. It declares the dimension it belongs to, the paths it applies to, and what it requires or conflicts with. Each dimension contributes its packs independently. Only rare intersections (queues inside a DDD project, for example) need a pack of their own.
2. **Manifest.** A short file declares the traits of the project. It is the only hand-written part.
3. **Detection that proposes, a person who approves.** A detector reads repository signals (dependencies in package.json, services in docker-compose, folder names) and suggests traits. It never writes new rules. Research on agent context files (ETH Zurich, secondhand figures) suggests LLM-generated context performs worse than curated context, so the agent selects from a reviewed library instead of inventing.
4. **Resolver.** It merges packs with a precedence order (personal, team, stack, project), reports conflicts at build time instead of during a session, and emits each tool's format: path-scoped rule files for Claude Code, an AGENTS.md section, Cursor rule files.
5. **Enforcement ladder.** A rule written as text is the weakest form, because agent memory is context and not enforced configuration. Each rule should move as high as it can: types or lint first, then a hook, then a test, and text only when nothing else applies. This is the same idea as retiring a spec once code replaces it, applied to rules.
6. **Budget and lifecycle.** A maximum number of rule lines loaded per task. Add a rule when the agent repeats a mistake, and remove a rule that never fires.
7. **Evaluation.** A small corpus of tasks lets us compare agent results with and without a rule. This is the method the ETH study used, and it answers a gap some rules admit about themselves ("whether it improves results has not been measured").

Example manifest:

```yaml
# rules.manifest.yaml
facets:
  language: [typescript]
  architecture: [ddd, hexagonal]
  stack: [postgres, bullmq]
overrides:
  disable: [code-english-only]
budget_lines: 200
```

## Pack metadata (frontmatter)

Claude Code reads only the paths field of a rule and ignores every other field without an error, so extra metadata is safe today and useful for a future resolver. Fields we would use:

| Field | Meaning |
|---|---|
| paths | Files the rule applies to. Omit for a global rule. Read by Claude Code. |
| tier | universal (any project), architecture (a style) or stack (a technology). Decides where the pack lives and its precedence. |
| facets | Traits that activate the pack, for example any, ddd or queue. |
| enforce | How the rule is enforced today: text, lint, hook or test. |
| upgrade | The strongest mechanism it could move to: lint, hook, test or none. |
| evidence | unmeasured or measured, once an evaluation exists. |
| version | Integer, raised when the meaning changes. |
| requires, conflicts | Other packs this one needs or cannot coexist with. Only written when non-empty. |

Example, as used by the domain-layer rule in the webhooks project:

```yaml
---
paths:
  - "**/core/**"
  - "**/domain/**"
tier: architecture
facets: [ddd]
enforce: text
upgrade: lint
evidence: unmeasured
version: 1
---
```

## What exists and what is missing

- **Syncing one rule set to several agents:** tools such as Ruler write CLAUDE.md, Cursor rule files and others from one source folder.
- **Distributing and versioning skills, hooks and agents:** Claude Code plugins and marketplaces.
- **Not found:** choosing packs from the traits of a project, resolving conflicts between them, and measuring whether they help. This is the gap the idea would fill. It is an impression from two searches, not a full market study.

## Risks

- Over-engineering: a rule library is a product that needs maintenance.
- Security: third-party packs may include hooks or shell commands that run on a developer machine, so every external pack needs review.
- False sense of control: more rules do not guarantee the agent follows them.

## Minimal path to try it

1. A folder of packs in a git repository, and a script that copies the packs a manifest names. No detection.
2. Add detection that proposes traits.
3. Add a conflict check and the line budget.
4. Add evaluations.

First cheap experiment, inside the webhooks project: split the existing rules into universal, architecture and stack tiers, and add one agnostic stack rule (queue handling).

## Open questions

- Is a manifest needed, or is detection plus an override file enough?
- Where does the library live: its own repository, or a plugin?
- How are conflicts expressed so that a resolver can check them mechanically?
- What is the smallest evaluation corpus that gives a trustworthy signal?

## Sources

- Ruler README: https://app.unpkg.com/@intellectronica/ruler@0.3.10/files/README.md
- Claude Code plugins reference: https://code.claude.com/docs/en/plugins-reference
- Claude Code memory and path-scoped rules: https://code.claude.com/docs/en/memory
- ETH Zurich, Evaluating AGENTS.md: https://arxiv.org/abs/2602.11988v1
