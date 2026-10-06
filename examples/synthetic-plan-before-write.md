# Synthetic example — plan-before-write (implementer assistant)

> **Synthetic example inspired by the production workflow.** The scenario is fabricated. The *shape* of the plan card and the per-tool approval UX is faithful to the real product.

## Scenario

An implementation specialist is onboarding a new business entity for a client — a "Delivery slots" catalog. The implementer assistant is open in the admin panel.

The specialist types, in Romanian:

> *Avem nevoie de o nouă nomenclator "Delivery slots" cu coloanele: Code (text, unique), Name (text, translated), ActiveFrom (date), ActiveTo (date, optional), MaxCapacity (number). Să fie accesibilă din meniul "Operations".*

The agent does **not** act. It responds with a plan card.

---

## The plan card (SPA rendering, conceptual)

```text
Plan proposed — please review and approve.

Step 1 — create_table [write]
   Table: DeliverySlots
   Metadata flags: masterdata=true, hasAudit=true, hasTranslations=true
   Columns:
     - Code       (nvarchar(50), unique, descriptor)
     - Name       (nvarchar(200), translated)
     - ActiveFrom (date, not null)
     - ActiveTo   (date, nullable)
     - MaxCapacity (int, not null)
   [ Approve this step ]   [ Cancel this step ]
   [ Show raw JSON ]

Step 2 — create_window [write]
   Window: DeliverySlotsList
   WindowType: grid
   Bound to table: DeliverySlots
   Fields shown: Code, Name, ActiveFrom, ActiveTo, MaxCapacity
   [ Approve this step ]   [ Cancel this step ]
   [ Show raw JSON ]

Step 3 — create_menu_item [write]
   Menu: Operations > Delivery slots
   Icon: dx-icon-event
   Opens: DeliverySlotsList
   [ Approve this step ]   [ Cancel this step ]
   [ Show raw JSON ]

Step 4 — sync_table_to_db [write]
   Table: DeliverySlots
   Purpose: physically create the SQL table from the metadata rows.
   [ Approve this step ]   [ Cancel this step ]
   [ Show raw JSON ]

[ Approve all ]     [ Cancel entire plan ]

Auto-approve read-only steps:  ☐  (off)
```

The user sees the plan in Romanian, the way the UI is localized. The structure of each step is the same regardless of language: a short human-readable summary, the tool name, the raw arguments collapsed by default, and three buttons.

## What happens if the user approves

The server executes the approved steps **in the same transaction**, through the typed persistence layer. Hooks fire. Audit columns get stamped. Translation rows are auto-created for the `Name` column. The capture pipeline writes one migration envelope attributed to the user and their currently-selected task.

The agent is told the result:

```text
Step 1: created table DeliverySlots (metadata rows: table + 5 columns)
Step 2: created window DeliverySlotsList (metadata rows: 1 window + 5 fields)
Step 3: created menu item under Operations
Step 4: synced to DB — physical table created with 5 columns + 4 audit columns
```

The agent then produces its final answer:

> *Catalogul "Delivery slots" a fost creat și e accesibil din meniul Operations. Următorul pas: adaugă în el valorile inițiale (dblclick pe grilă sau folosește importul din Excel).*

## What happens if the user cancels step 4

Step 4 depends on steps 1–3 for its inputs (the metadata rows that it will materialize), but it is the last step in the chain — nothing in the plan depends on it. The server executes steps 1–3 and skips step 4. The resulting state is partial: the metadata rows exist, but the physical table has not yet been created from them. The user accepts this partial state when they cancel step 4; the correct next move is to run the materialize step later (manually, or by re-asking the agent), when they are ready.

The conversation state keeps the cancelled step as "cancelled by user" so the agent does not retry it silently in a later turn. The agent is told the partial outcome and either continues (asking for clarification on why step 4 was skipped) or produces its answer with the known state.

## What happens if the user cancels a step that other steps depend on

> **Example-level assumptions below.** The three rules in this section describe what a plan-before-write UX *should* enforce to be safe; they are stated here so the trade-offs of the design are explicit. Whether the production implementation enforces each of them verbatim, or relies on narrower checks, is an implementation detail outside the scope of this synthetic example.

Suppose instead the user cancels step 1 (create_table) but still approves step 2 (create_window, which references the table that would have been created). The ideal behavior:

- **Rule 1 — ordered approvals.** A step cannot execute if any step it depends on was cancelled in the same plan. Approving step 2 after cancelling step 1 is rejected with "step 2 depends on a cancelled step; cancel step 2 or re-propose."
- **Rule 2 — dependency inferred from arguments.** The dependency graph is derived from the tool arguments (step 2's `table` argument names the output of step 1), not from the agent's self-report. This matters because the agent can describe a plan ambiguously, but the arguments are concrete.
- **Rule 3 — arguments are frozen after approval.** Once approved, a step's arguments cannot drift. If the agent would re-propose the same step with different arguments in a follow-up turn, that is a new plan card. This prevents the "I approved X, but X_slightly_different ran" class of defect.

If the user wants to run only step 2 against a pre-existing table, that is a different plan — the agent re-proposes with a plan that does not include step 1.

The rules above describe what the design aims for. A production implementation may enforce them in different places (server-side rejection vs. UX pre-check vs. agent-level self-restriction), with different trade-offs between strictness and usability. The point of this synthetic example is to make the design intent legible, not to publish the implementation.

## What happens if the user cancels the entire plan

The server runs nothing. The agent's conversation history is **rolled back to the last user turn** so the Anthropic tool-use / tool-result contract stays intact — half a plan dangling in the history would corrupt the next turn.

The user sees: *"Plan was cancelled. Try phrasing the request differently or asking me to explain what each step would do."*

## Why this UX exists

Three distinct failure modes it blocks:

1. **Hallucinated plan.** If the agent proposes a table with the wrong columns or a menu item in the wrong tree, the user sees it before it runs.
2. **Silent scope creep.** The plan card shows every tool the agent would call. "I also want to run a sync" is visible, approvable, and rejectable individually.
3. **Replayed plan.** The server tracks the plan state; a user cannot be tricked into clicking "approve" on a plan that mutated between proposal and execution.

## Why read-only steps have their own toggle

Some tools are safe by construction — list tables, describe a table, run a vetted `SELECT` through the SQL guard, retrieve from the knowledge base. For a fluent conversation, requiring approval on each of those is friction without safety value.

The auto-approve-read-only toggle moves read-only tools through automatically; writes always surface. The default is off. The decision is per-user and persistent.

## What this is **not**

- Not an autonomous agent. The user is on the critical path for every write.
- Not a dry-run / preview. The server does not execute speculatively.
- Not a replacement for role-based access. The implementer role still gates the entire panel; the plan card is in addition, not instead.

## The one line of design that makes this safe

> **Approve/cancel is at the tool-call level, not the plan level.**

A plan-level approval would be too coarse; the user would either accept a dangerous step they did not notice or cancel a correct plan over one bad step. Per-tool approval is granular enough to let the user keep the good steps and drop the bad one — which is the behavior that matches how experienced implementers actually think.
