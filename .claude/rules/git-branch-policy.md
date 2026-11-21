# Branch Name Conventions

This governs the name and lifetime of every git branch created in this repository, by a person or by Claude. It applies globally, so it carries no `paths:` scope. Commit messages and pull request titles are out of scope.

---

## Name

- A branch is named `<type>/<slug>`. The type comes from the same vocabulary as commit types (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`), because a branch and the commits on it describe the same kind of change, and tooling can group both the same way.
- The slug is lowercase ASCII words joined by hyphens, two to five words, describing the outcome and not the author, the date or the status: `add-retry-policy`, not `wip-juan` or `fix-stuff`. The whole name stays under 50 characters so it fits in lists and terminals.
- A project that delivers in numbered plan phases may name a branch `phase-<n>-<slug>` instead, when its own documents say so. The phase number then replaces the type.
- No spaces, uppercase letters or underscores, and no slash beyond the single type separator, because they break shells, URLs and tooling.

## One purpose, short life

- One branch carries one logical change, in the sense of the commit rule: a phase of work, not its smallest step. Unrelated work gets its own branch, because a branch has to be reviewable and revertible as a unit.
- A branch starts from the up-to-date default branch and is short-lived: merge or discard it as soon as the work is checked, because long-lived branches drift and turn merges into projects.
- Claude may create and switch branches when the work calls for it. Pushing and pulling stay denied by the permission policy.

---

## Non-goals

- Doesn't dictate commit messages or pull request titles.
- Doesn't decide when work deserves a branch instead of a direct commit on the default branch.
