# Case study — WiData / WiData Inventory

> A real, in-production application built on the earlier generation of the framework described in this repository. Public presentation attached; everything in this document is non-confidential and reflects what has been publicly shown by the WiData product line.

## 1. In one line

A warehouse management platform built on a metadata-driven engine I architected and developed between **2014 and 2026**, in production since **2015**, running at **tens of customers**, including an operation with **more than 10,000 users** on a single deployment.

Public reference: [WiData Inventory presentation on YouTube](https://www.youtube.com/watch?v=8Oi9CupwpsY).

## 2. What WiData is

WiData is an enterprise platform for operations software in the warehouse-management vertical, built in a way the industry does not usually attempt: the application behavior is driven by metadata rather than per-entity code. Adding a new business entity — a new catalog, a new document type, a new workflow — is a configuration change on the running system, not an engineering release.

WiData Inventory is the inventory-and-warehouse-operations application built on top of that engine. It covers the operational flow a warehouse business needs: inbound, storage, picking, replenishment, outbound, inventory counts, and the reporting on top. Different customers run different configurations; the engine is the same.

## 3. The scale that matters

A few numbers that are relevant to this case because they say what the architecture had to hold up to:

- **In production since 2015** — eleven production years at the time of writing.
- **Tens of customers** on the same codebase, with per-customer configuration rather than per-customer forks.
- **A single deployment with 10,000+ users** — the architectural upper bound for the design, hit in production, not in a slide.
- **Vertical:** warehouse management; operations-grade uptime expectations.

These numbers matter because they rule out the easy objection to metadata-driven designs — "it works until it meets production load". It met production load, repeatedly, for a decade, on diverse customer contexts.

## 4. The engineering problem the architecture solved

Enterprise operations software suffers from a predictable decay: each new customer requires a slightly different variation of the same abstract capability (receive goods, pick goods, count stock, reconcile), implemented slightly differently by slightly different people. Over a few years the codebase becomes a drift of per-customer forks. Shipping a cross-cutting change — a new audit requirement, a new permission model, a new report format — has to touch every fork.

The metadata-driven engine collapsed that fan-out. The capabilities are generic; the per-customer differences live in registry rows (tables, columns, grids, forms, actions, translations, roles). A new customer onboarding is a configuration pass, not an engineering project. A cross-cutting change ships once, in the engine.

The structural choices that made this possible:

- A small set of registry tables describing what the application contains.
- A generic runtime (row/repository/cache) that reads the registry at startup and services every request from it — no per-entity controller, no per-entity repository, no per-entity UI component.
- Logical schemas resolved at runtime, so a legacy database that already carries one of those schema names can be onboarded without renaming anything.
- A migration-capture pipeline where real admin edits on the reference environment become deployable envelopes, replacing the "hand-written SQL migration" model that cannot survive many customers.

Those four choices are the lineage that carries into the framework documented in this repository.

## 5. What carried into the current framework

The current framework (built on React + .NET + Azure) is the successor to WiData's engine. The ideas carried forward are the ones that survived a decade of production exposure:

- **Metadata is the application.** The engine reads what tables, columns, grids, forms, FKs, translations exist; it does not hard-code any of them.
- **Logical, not physical, schemas.** Different deployments can place the same logical schema in different physical places.
- **Capture, don't hand-write.** Admin edits on the authoring environment become envelopes; customers apply them. No hand-written SQL migrations.
- **Live replication inside the capture pipeline.** A change made once on the reference environment reaches the registered test deployments in the same transaction — the "I made it here but forgot there" window collapses to zero.
- **Hooks, not triggers, for business logic.** Business rules live in code that is readable from the application; triggers are reserved for infrastructure (audit, replication, guard).

Each of these was learned the expensive way in the WiData era and is now a hard rule in the current framework.

## 6. What is new in the successor

The current framework is not a port. It is a rebuild on a current stack, with ideas added that did not exist in the WiData era:

- **An AI-native development model.** A coordinator with specialized L1 and L2 verifiers; mechanical traceability per task; a fail-closed AI reviewer on CI. Documented in [agentic-build-workflow.md](agentic-build-workflow.md) and [ai-native-development.md](ai-native-development.md).
- **In-product AI assistants.** Two of them, with the distinct trust models explained in [in-product-ai-agents.md](in-product-ai-agents.md).
- **A modern SPA stack.** React + TypeScript on the client, replacing the earlier Razor + jQuery front-end. Same metadata contract, different rendering.
- **Cloud-native deployment.** Azure App Service and VMs, Key Vault for secrets, DevOps-based CI/CD, replacing the on-premise model that WiData originally targeted.

What stayed is the engine. What changed is everything around it.

## 7. Why this case is worth citing

A pattern argued only in the abstract is weak evidence. A pattern that has been in production for over a decade, at the scale WiData reached, is a different kind of argument. The lineage says that the metadata-driven approach described in this repository is not a design idea; it is a proven architecture, re-expressed on a current stack with current tools, by the same architect, with the lessons from the first decade incorporated.

That is the point of including this case here: to show that the ideas in the other documents have been tested, in production, for a decade, before being re-built in the current form.

## 8. What is deliberately not here

- Customer names.
- Specific warehouse workflows that are proprietary to WiData's customers.
- Internal naming or identifiers from the WiData codebase.
- Specific numbers beyond what is public.

The YouTube presentation is the public reference. This document is an engineering narrative, not a product datasheet.
