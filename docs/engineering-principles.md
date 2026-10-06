# Engineering principles

> A distilled, public-safe version of the canonical principles used on the real project.
>
> The real document is longer, versioned in the repo, and read by every build-time agent at the start of every session. The rules below are the ones a senior engineer visiting this repository would recognize as the backbone — mechanical where possible, enforced by a reviewer or a hook where not.

## 1. The engine, not the entity

Code must be generic. If a change requires a per-table controller, a per-table repository, or a per-table React component, something upstream in the engine is wrong. Fix the engine. The surface area of the application grows through metadata rows, not source files.

## 2. Read the code before writing code

Any non-trivial change starts with reading the specific files it will touch and the mechanisms adjacent to them. "I will add a cache here" is forbidden if a cache already exists elsewhere; "I will introduce a new helper" is forbidden if an existing helper covers the case. Reusing existing mechanisms is a rule, not a preference.

## 3. No raw SQL in business code

Business operations go through the typed `Row` + `Repository` layer. Dirty tracking, hooks, audit, translations, access control, and capture all live there. Bypassing it is a correctness problem, not a style preference.

Raw SQL is accepted inside a short list of categories (DDL, system catalog, dynamic table name, cross-DB, atomic upserts on natural keys, atomic increments, etc.), and only with an inline `// RAW-SQL-OK: <reason>` comment. Any raw SQL without a comment or without an accepted reason is a review stop.

## 4. Hard-coded identifiers are forbidden

Primary key names, audit column names, descriptor column names, FK conventions — none of these are hard-coded in business code or in the SPA. The metadata layer is the source of truth for every one of them. One central fallback is permitted; scattered fallbacks are not.

The reason is specific: the primary key is not always `ID`. On a legacy or client-side table it can be `Id`, `id`, or something else entirely. Hard-coded `"ID"` works coincidentally on one database and breaks silently on another.

## 5. All user-visible text goes through the translation layer

Every string that reaches a user — labels, buttons, tooltips, placeholders, error messages, exception text — is a translation key resolved through the translation layer. The sole exception is operational data that comes from business tables.

New keys are added as database rows per language, not as string constants in code. Localization is not a style choice; it is the product's international surface.

## 6. One action, one transaction

Any action (UI click, API call, scheduled job) that touches the database runs on a single new transaction. All hooks, all adjacent reads, and all sub-operations share that transaction. Success commits; any error on the path rolls back the whole thing.

Hooks never open new connections. They never spin threads. They never use `.GetAwaiter().GetResult()` on an async hook. Transactions are thread-bound; code that forgets that produces subtle, hard-to-diagnose bugs.

## 7. No silent catches

A `catch` block either re-throws, or logs and throws, or surfaces the error to the user. Never an empty catch. Never "ignore and continue".

Silent catches are the dominant source of hidden defects in enterprise codebases. The rule is strict because the failure mode is quiet.

## 8. No logic in triggers

Triggers are infrastructure (audit capture, replication, guard), never business logic. If a rule needs to run on write, it runs in a hook, not a trigger. Business logic in a trigger is invisible from the code the user is reading and breaks when sync is involved.

The only accepted business-side trigger is a narrow production-only guard, and the general direction is to remove even those.

## 9. No ALTER / CREATE / DROP without consent

Schema changes are proposed and executed only after explicit human approval. The proposal must include the column, type, table, and purpose. "I think we need a column here" is not a schema change; "please add column X of type Y to table Z, which will carry the following" is.

Schema changes propagate through the capture pipeline. They are never hand-written SQL files.

## 10. Audit columns are mandatory

Every table carries four audit columns: insert-date, insert-user-id, last-change-date, last-change-user-id. The engine stamps them automatically. Omitting them, or renaming them per table, defeats the auditability guarantee the product offers.

## 11. The reference environment is where masterdata is authored

Masterdata is authored once, on the reference environment, by a human with a selected task. The capture pipeline produces an envelope; customer environments apply it. Hand-editing masterdata on a customer environment is not supported — it is explicitly guarded against.

A corollary: local development never writes to the production reference environment. Specific scripts enforce this; a mistake on this axis would cross a security boundary.

## 12. Never remove features without explicit confirmation

Deletion is only on explicit human request. If a feature has a bug, fix the bug; do not delete the feature. If a seemingly-unused piece of code may still be referenced, verify before removing. The failure mode of over-eager deletion is silent and expensive.

## 13. Suggest structural changes; don't execute them

A better schema, a better abstraction, a better directory layout — the correct move is to suggest, not to perform. The engineering value of consistency is high enough that unilateral restructuring is worse than a slightly-imperfect current state.

## 14. Traceability per task is mandatory

Any non-trivial change writes three files, in order:

- **01-Requirements** — the user's text verbatim, plus agreed clarifications.
- **02-Solutions** — design, dispatch plan, risk, and a **verbatim approvals** section for every decision that changes functionality.
- **04-Tests** — the evidence that the thing works, with artifacts attached to the ticket.

A pre-commit hook refuses to commit touched code without 01 and 02. A pre-push hook refuses to push without 04. A CI reviewer rechecks the whole trio against the diff and refuses the PR on mismatch.

## 15. Principles are not skipped

Any violation of a principle is fixed on the spot, sweep-wide. Not "later", not "only the instance that is blocking me". Partial enforcement is worse than none — it creates a precedent that principles are negotiable.

A trigger phrase on the human side (used in Romanian in the real project) prefixes a task with "watch the principles". That is a checklist obligation to re-read the principles document top-to-bottom before declaring the work done.

## 16. Diagnosis before fix

Reporting that a bug has a known cause is a two-step discipline: investigate first (read the actual code, follow the real call path, verify against the files rather than against memory of how it "should" work), then communicate the cause with file and line references.

Fixing before diagnosing is forbidden because speculative fixes either mask the real bug (symptom moves, cause persists) or add new bugs on top of the old one. Diagnosis-then-fix is slower per step and faster overall.

## 17. Verify presence of the thing before pointing to it

Before recommending a function or a flag that memory says exists, verify it is still there. Memory is a snapshot; the code moves. "The memory says X exists" is not the same as "X exists now". Grep, read, confirm.

## 18. Push on every commit; never auto-open a PR

Each commit is pushed to its branch. Pull requests are opened only when the human explicitly asks. Automatic PR creation is a form of unilateral commitment the human did not approve.

## 19. Explicit "go" before any modification

Even when the diagnosis is clear and the user has given a general direction, the plan is presented explicitly and the engineer waits for a `go` before executing. Plan-then-execute reduces the total back-and-forth. A plan without a plan is more talking, not less.

## 20. English in code, local language in docs

Code comments and commit messages are English. User-facing documentation and translations include the local language. The real project maintains both languages in parallel for every functional change; a change that updates one but not the other is blocked.

---

These are not aspirations. Each one has a mechanical enforcer (hook, reviewer, or CI gate), a documented reason, and a history of incidents that justify it.
