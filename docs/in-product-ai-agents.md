# In-product AI agents

> The AI inside the shipping product. Two assistants, two very different trust models. One of the architectural ideas this repository is meant to communicate.

## 1. Why this distinction matters

The AI used to **build** the product and the AI **shipped inside** it are different systems with different accountability. Collapsing them into "the AI feature" is a category error that leads to the wrong permissions, the wrong guardrails, and the wrong trust assumptions.

A short recap of the comparison, repeated here because it is the single most important idea for a reviewer to carry away:

| Dimension | Build-time agents | In-product agents |
|---|---|---|
| Author of the output | engineer dispatching agents | an end user or an implementation specialist clicking |
| Trust model | reviewed by other agents + human + CI | enforced by product-layer permissions + plan-before-write UX |
| Accountability | the engineer who merged the PR | the user who approved the write |
| Primary risk | plausible-but-wrong code that merges | arbitrary data write or information leak |
| Failure containment | adversarial review + fail-closed CI | role-gated catalog of tools + per-write human approval |

The rest of this document is about the right column — the two agents that ship inside the product.

## 2. The two assistants

Two agents share the UI shell but have completely different internals.

### End-user assistant

- A drawer in every screen. Opens on demand, closes to a thin edge.
- Answers questions about the product in the user's language.
- Grounded on an indexed knowledge base (product docs + concept reference).
- No tools. No writes. No actions. If retrieval does not clearly answer the question, it refuses rather than guesses.
- Backed by a small, fast model — latency and cost predictable, context small.
- The topology has two processes: the main application server exposes a thin proxy endpoint (so the SPA keeps a single origin and inherits authentication), and a separate RAG microservice owns ingestion, chunking, embedding, retrieval, and the actual ask endpoint. The proxy forwards authenticated requests to the microservice; the microservice never faces the browser directly.

### Implementer assistant

- A dedicated admin panel for implementation specialists working on a deployment.
- Has a curated catalog of tools that goes through the **same typed persistence layer the UI uses**, so permissions, audit, hooks, and translations fire exactly as if a human admin were clicking.
- Every write surfaces a plan card. The user approves or cancels each tool call before anything is applied.
- Backed by a frontier model — needs strong tool-calling and planning.

Both agents log every turn to an append-only audit table.

## 3. Permissions

Access to each assistant is scoped by role:

- **End-user assistant.** A role flag grants access to the drawer. The reason the gate exists is that answers can include session-contextual information the user already has access to; the role controls whether that enrichment is on.
- **Implementer assistant.** A dedicated implementer role, plus an alternative gate for full-access admins. No bypass by API key alone — the credential must be bound to a user with the right role.

Credentials:

- A per-user API credential is stored hashed in a dedicated table (one-way, with a strong hash).
- A shared provider API key is stored in the secrets vault; the application reads it via managed identity. Nothing lands in config files or git.

## 4. Guardrails

### SQL guard

The implementer agent has a read-only SQL tool. That tool goes through a dedicated guard that:

- strips comments and literals before analysis so comment-based bypasses do not work;
- rejects multiple statements per call;
- requires the first keyword to be `SELECT` or `WITH`;
- rejects anything on the deny list — `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `CREATE`, `EXEC`, `MERGE`, `TRUNCATE`, `DBCC`, `WAITFOR`, and friends;
- restricts the referenced schemas to a short, configured allow-list (the framework's own schemas plus the standard SQL Server catalog schemas);
- enforces a command timeout and a row cap, with an absolute ceiling that cannot be raised by prompt.

Important caveat: the guard is a **syntactic filter**, not an authorization layer. It prevents a syntactically harmful statement from reaching the database, but it does not substitute for row-level access control, column-level security, or database-user privilege scoping. The guard assumes the connection it runs on already has the appropriate least-privilege configuration; without that, a syntactically-valid `SELECT` can still over-read. Authorization lives one layer below.

### Grounding floor (end-user)

The RAG retriever has a minimum relevance score. Below that floor the agent is told "no relevant context" and refuses — it does not improvise. This is the single most important defense against hallucinated answers that reference the product's own vocabulary.

### Plan-before-write UX (implementer)

The implementer agent never applies a write directly. The flow is:

1. The agent proposes a plan — a list of tool calls, each with a short human-readable summary in the user's language, the tool name, and the raw arguments collapsible for review.
2. The user approves, cancels individual steps, or cancels the entire plan.
3. The server executes the approved steps and returns results to the agent.
4. The agent either continues (with another plan card) or answers.

The approve/cancel UX is the primary safety mechanism. It is backed by state on the server so a plan cannot be "snuck through" by replaying a request.

### Attachment caps

Both agents accept attachments. Caps are enforced server-side, mirrored client-side:

- a maximum number of files per turn
- a per-type size cap (images, PDFs, text) and a total-turn cap
- a MIME allow-list — anything else is rejected before it leaves the browser

Attachment bytes are **never** persisted into the audit log. Only filename, size, and MIME type.

### Audit

Every turn, for both agents, writes one row into an append-only audit table:

- user, session, language
- the question text and the response text
- the list of tool calls (name, success, duration), as structured JSON
- token counts for prompt and completion, plus cache-read and cache-create counts
- an error flag and a start/stop UTC timestamp

This is reviewable: implementation managers can and do read what their teams are asking the agent to do.

Known limitations of the audit as described, deliberately listed so they are not confused with claims of safety:

- **No field-level redaction.** Question and response text are stored as the user wrote and the model produced. If a user pastes a secret into the question, it lands in the audit as-is. Mitigation is at the policy layer (users are told not to paste secrets), not at the storage layer.
- **No automatic retention policy at the application level.** Retention is governed by the deployment's broader data policy; the audit table itself does not expire or auto-prune. Deployments that need an expiration must configure it externally.
- **Access control is role-gated, not row-gated.** A user with the audit-read role sees all rows; there is no per-user filter at the storage layer. For most deployments this is correct (managers see their team); deployments that need stricter isolation must add a view with filtering.
- **Provider egress.** The question text and the response text are sent to the LLM provider by construction (that is how the model works). The audit records what left the application; it does not add containment. Deployments with strict data-residency constraints must evaluate the provider boundary separately.

These are documented so the audit is not read as "safe by design". The audit makes agent use *reviewable*, which is a different guarantee than *contained*.

## 5. Tooling (implementer agent)

The tool catalog is small on purpose. Each tool goes through the same typed persistence layer as the UI, so all the UI-level invariants apply automatically.

Groupings (names are illustrative, not the real names):

- **Read-only.** List tables, describe a table, list windows, describe a window, list menus, run a vetted `SELECT` through the SQL guard, retrieve from the product doc index.
- **Curated writes.** Create a table, add a column, edit a column metadata row, create a window, add a field to a window, create a menu item, create a dictionary and add values, configure an API service, run a sync-to-DB operation for a given table.
- **Generic primitives.** A row upsert, a row delete, a bulk batch (transactional), an invocation of a registered action, a whitelisted system operation (cache reset, cache reload, FK revalidation).

Every tool call lands in the audit log with its full argument set. The implementer sees exactly what was proposed, exactly what was approved, and exactly what ran.

## 6. Model routing

- End-user assistant: a small, fast model. Grounded RAG, short response ceiling, latency-sensitive.
- Implementer assistant: a frontier model. Needs multi-step planning and tool calls, longer response ceiling, correctness-sensitive.
- Embeddings: a dedicated provider for the knowledge base.

Prompt caching is on. The system prompt and the recent history use cache breakpoints to amortize cost across a conversation. The caching strategy is important enough on cost and latency that it is treated as a product feature, not an infra detail.

## 7. Where human oversight stays mandatory

The two assistants are useful inside their lanes. Outside them, humans still own the decision:

- **Writes that touch permissions or security.** The implementer agent will not propose, and the UI will not accept, a tool call that silently alters access control. Those changes go through the metadata UI, with a human in front of them.
- **Writes that cross application boundaries.** A multi-application change is split into separate tasks by a human, not merged by the agent.
- **Writes on production shared data.** The implementer agent is used on reference and test environments. Running it against production data would require explicit role grants that are not given by default.
- **Interpretations the user did not ask for.** The agent is instructed to refuse when retrieval does not clearly answer; the UX makes refusal the easy path. "I don't know" is not a failure mode here, it is the correct answer.

## 8. What this architecture buys the product

- **An assistant that refuses rather than invents.** The end-user drawer stays useful because the grounding floor pushes refusal when retrieval is weak — reducing the surface where a user acts on a plausible answer that is not supported by the docs. It does not eliminate incorrect answers; it narrows the conditions under which they appear.
- **An implementer that proposes before it writes.** The plan-before-write UX turns the implementer agent into a reviewed-by-default collaborator rather than an autonomous rewriter. It reduces blast radius; it does not remove the need for the user to read the plan.
- **A reviewable record of AI use in production.** The audit table lets managers see what their teams are doing with the agents — a prerequisite for giving them real work. Reviewable is not the same as contained (see the audit limitations above); it is the layer that makes containment decisions possible.

## 9. Why these two agents are not one agent

Collapsing them would require either:

- promoting the end-user agent with tools and writes — and losing the "no writes, limited blast radius" posture that makes the drawer easy to trust; or
- demoting the implementer agent to read-only — and losing most of its usefulness.

(No writes still leaves other risks — primarily information surfacing. Grounding floor and refuse-when-unknown contain that risk; they do not eliminate it.)

Keeping them separate keeps each one's trust model simple. The end-user agent is a grounded reader. The implementer agent is a plan-and-execute operator. Each one is understood by its users on exactly those terms.

## 10. What is deliberately not here

- Specific prompt bodies, personas, or persona wording.
- Specific model names and token counts.
- Specific tool JSON schemas, which would reveal the metadata layer.
- Specific role names and permission flag identifiers.
- Specific audit-table column names.

The patterns above are faithful. The implementation details are intentionally omitted.
