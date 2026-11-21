# Rules, Playbooks and Context: idea for a separate project

> **Status:** parked idea, partly tried in the webhooks project. Written down so it is not lost. Nothing in webhooks depends on it.

## The problem

Rules that constrain a coding agent depend on the kind of project. Variability comes from independent dimensions: the stack (queues, databases, HTTP frameworks), the architecture style (DDD, hexagonal, plain MVC), the domain, the maturity of the project, and team or personal conventions.

One rule file per combination explodes quickly. Copying rules between repositories makes them drift. Loading everything in every session costs tokens and lowers adherence. And a rule that mixes "what must hold" with "how to do it in technology X" is neither short nor reusable.

## The idea: separate the constraint from the how

Three containers, each with one job and a different loading mechanism.

| Container | Holds | Loaded | Size |
|---|---|---|---|
| **Rules** | Behavior constraints, agnostic of any technology | Always, or by path | A handful |
| **Playbooks** (one skill) | How to do it in a given technology: queues, outbound HTTP, SQL, signing, testing | On demand, when the task needs it | One skill, many references |
| **Context** | Facts and decisions of one project | When the agent reads it | One folder per project |

"Jobs must be safe to run twice" is a constraint, so it is a rule. "How to make a job idempotent in a given queue library" is a manual, so it is a playbook reference. The manual does not need to sit in context until the task touches that technology.

### Rules

Small, imperative, each bullet with its reason. Written so they hold in any project. Tagged by tier (universal, architecture, stack) and scoped by path when they only matter in some folders. Current set in webhooks: lean change, English only, commit messages, branch names, domain layer, edge safety.

The edge safety rule is the example of compaction. Instead of one rule per technology (queues, outbound HTTP, data access, signing, logging, configuration) it states six points that hold at every boundary: validate on the way in, bound time and size, make repeats safe, translate failures, keep survivable state in the system of record, never log or return secrets.

### Playbooks

One skill whose description lists the technologies it covers. Only that description is always visible to the agent; each reference is read when the task needs it:

```
stack-playbooks/
  SKILL.md          # short description: guidance for queues, HTTP, SQL, signing, tests
  references/
    queues.md       # opened only when the task touches queues
    http-egress.md
    sql.md
```

A playbook is **created when a project needs it, not before**. When the agent or a developer hits a technology with no reference, a draft is written from the official documentation and the project's experience, a person reviews it, and then it is stored and reused. It is generated once and curated, never regenerated on every session: research on agent context files (ETH Zurich, secondhand figures) suggests unreviewed generated context performs worse than curated context.

Skills are matched by their description, so a playbook can fail to trigger. Anything that must always hold stays in a rule, and playbooks only add guidance.

### Context

Project facts and the reasons behind decisions: what no rule or playbook can know. In webhooks it is the context folder.

## Where things live for reuse

| Tier | Example | Location |
|---|---|---|
| Universal | Commits, branch names, language of code, lean change | User-level rules, shared by every project |
| Architecture | Domain layer, edge safety | Project rules, or a plugin shared by projects of the same style |
| Stack | Technology playbooks | The skill, distributed as a plugin |

## Rule metadata (frontmatter)

Claude Code reads only the paths field of a rule and ignores every other field without an error, so extra metadata is safe today and useful to a future tool. The webhooks project keeps only paths for now, because it is small. The fields below are the set to adopt when a tool needs them.

| Field | Meaning |
|---|---|
| paths | Files the rule applies to. Omit for a global rule. Read by Claude Code. |
| tier | universal, architecture or stack. Decides where it lives. |
| facets | Traits that activate it, for example any, ddd or hexagonal. |
| enforce | How it is enforced today: text, lint, hook or test. |
| upgrade | The strongest mechanism it could move to, or none. |
| evidence | unmeasured or measured, once an evaluation exists. |
| version | Integer, raised when the meaning changes. |

## Enforcement ladder

Text is the weakest form, because agent memory is context and not enforced configuration. Each rule should climb as high as it can: types or lint, then a hook, then a test, and text only when nothing else applies. This is the same idea as retiring a spec once code replaces it.

## Earlier variant, parked

A library of packs with a manifest, a detector that proposes traits and a resolver that compiles the matching packs for each agent. It is heavier, and only worth it if the library grows past what the three containers handle. Ruler already syncs one rule set to several agents, and Claude Code plugins distribute skills and hooks. Nothing found chooses packs by project traits, resolves conflicts or measures benefit. That impression comes from two searches, not a market study.

## Risks

- Over-engineering: a rule library is a product that needs maintenance.
- Security: third-party skills or plugins may include hooks or shell commands that run on a developer machine, so each external one needs review.
- A generic rule can become so general that it says nothing. Each point needs a reason, and ideally an example.
- More rules never guarantee the agent follows them.

## Minimal path

1. Rules in the repository, agnostic and small. Done in webhooks.
2. Create the playbook skill when the first technology need appears, with one reviewed reference.
3. Move the universal tier to user-level rules.
4. Package rules and skill as a plugin.
5. Later: evaluation on a small corpus of tasks, and the resolver if the library grows.

## Open questions

- When is a playbook promoted from a project note to a shared reference?
- How do we know a playbook was actually opened when needed?
- What is the smallest evaluation corpus that gives a trustworthy signal?

## Sources

- Ruler README: https://app.unpkg.com/@intellectronica/ruler@0.3.10/files/README.md
- Claude Code plugins reference: https://code.claude.com/docs/en/plugins-reference
- Claude Code memory and path-scoped rules: https://code.claude.com/docs/en/memory
- ETH Zurich, Evaluating AGENTS.md: https://arxiv.org/abs/2602.11988v1
