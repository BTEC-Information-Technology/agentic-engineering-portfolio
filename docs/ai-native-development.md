# AI-native development

> How AI is actually used in the engineering process behind the Platform. This document explains the philosophy; [agentic-build-workflow.md](agentic-build-workflow.md) explains the mechanics.

## 1. The central idea

> **AI should multiply engineering output without transferring engineering accountability to the AI.**

That one sentence drove every design choice below. The alternative — treating AI as an autonomous author of code — fails on real projects. An unconstrained LLM can produce plausible but architecturally wrong code faster than humans can review it, and the rate of plausible-wrong defects dominates everything else once a codebase passes a certain size.

The move that makes AI useful at engineering scale is to stop asking "can AI write this code?" and start asking **"what is the controlled engineering system around the AI?"**. If that system is right, AI genuinely multiplies output. If it is missing, AI is a liability dressed as a productivity gain.

## 2. "AI writes code" vs. "AI participates in a controlled engineering system"

| | AI writes code | AI participates in a controlled engineering system |
|---|---|---|
| **Instruction** | prompt + hope | written requirement, approved design, scoped context |
| **Context** | everything the agent can see | the specific slice relevant to the task |
| **Specialization** | one general-purpose agent | coordinator + specialized L1 verifiers + L2 fan-out |
| **Verification** | the agent says it's done | adversarial review by independent agents |
| **Tests** | the agent writes and asserts | the agent writes against human-reviewed assertions, run on real DB |
| **Review** | author reviews own work | author never reviews own work |
| **Human role** | typist | architect, reviewer, owner of the result |
| **Blast radius** | repo-wide | capped by the scoped context |
| **Failure mode when it goes wrong** | silent architectural drift at speed | failing gate before merge |

The right column is more work to set up. The payoff is that the fast-but-wrong failure mode — the one that otherwise dominates — is substantially reduced and, when it does occur, is contained to the gates designed to catch it before merge, rather than merging silently.

## 3. Principles come first, mechanically

The Platform has a canonical principles document — one source of truth, versioned in the repo. Every build-time agent reads it at the start of every session. Principles cover:

- what counts as raw SQL and when it is allowed
- when hard-coded identifiers are forbidden (and why)
- how transactions scope changes
- when hooks are mandatory and when they are a smell
- how translations and localization work end-to-end
- which schemas are logical vs. physical
- what belongs in the engine vs. in a per-entity module (short answer: almost nothing is per-entity)
- what the capture pipeline guarantees and what it refuses

An AI agent that drifts from a principle is wrong, regardless of how good the output looks. The adversarial reviewers verify principle conformance explicitly before anything is accepted.

## 4. Context is deliberately scoped

The most important mechanical correctness lever is context scoping.

A reviewer is given exactly the files they need to review and nothing else. A worker is given its branch and nothing else. A planner is given the task and the principles, but only reads code when the plan requires it. The coordinator's own context is kept small by delegating to subagents with cleanly bounded prompts.

Why this matters at the correctness level: beyond a certain context size, agents stop reading and start pattern-matching. They produce code that looks right against *the shape of their context* but is wrong against the specific file. Deliberate context scoping prevents this failure mode. The cost savings are a side effect.

## 5. Specialization

The build-time setup has roles, not generic agents:

- A **coordinator** that plans and dispatches.
- **L1 verifiers** per discipline — docs, front-end, back-end, tester, DB, communications. Each is responsible for one axis of correctness.
- **L2 verifiers** that fan out for adversarial review of a specific finding, in parallel.

Each role has a tailored prompt, a tailored tool set, and a tailored output format. The coordinator does not grep; the DB verifier does not touch SPA code; the tester does not approve its own tests.

Specialization matters because confirmation bias is the dominant LLM failure mode. An agent that writes code also tends to accept it; an agent that only reviews, with no stake in the code, finds more real defects.

## 6. Verification is adversarial, not confirmatory

Every finding goes through a two-stage review:

1. **Report.** A reviewer produces findings against a scoped slice of the change.
2. **Verify.** A second reviewer (or several, fanning out) takes each finding and tries to *refute* it: construct a concrete scenario where the claim is wrong, or confirm it with a failure trace. Findings that cannot be concretely refuted are reported; findings that fall over on verification are dropped.

Adversarial verification turns "AI said this was a problem" into "we have a verified failure scenario at file and line". The engineer looking at the report sees defects, not opinions.

## 7. Tests belong to humans

AI can draft tests. Humans decide what the tests must assert.

A test suite written entirely by an agent is dangerous because the agent asserts its own model of the world; the test passes by construction. On this project, tests are:

- scoped to the real database, not mocks, for anything that touches persistence
- assertions reviewed by a human before merge
- attached to the task as evidence (04-Tests file) before PR review proceeds

Green tests are not evidence of correctness on their own. Reviewed green tests, on the real integration surface, are.

## 8. Feedback loops

The system learns through:

- **The principles document.** When a reviewer catches a new class of defect that is not yet covered, it is added as a principle, versioned, and read by every subsequent session.
- **Memory per developer.** Each engineer accumulates a per-user memory of their preferences and corrections, loaded into every session. The project itself has a shared `MEMORY.md` that collects observations about the system — pending gaps, pitfalls, non-obvious invariants.
- **Lessons-learned.** Non-obvious failures are written up and referenced in the engineering docs so the whole team benefits.

The system does not retrain anything; it accumulates reviewable artifacts.

## 9. What stays human, always

Even inside a well-controlled AI system, specific decisions never leave the engineer:

- **Architectural boundaries.** Agents can refactor within a module; they cannot move a boundary. Introducing a new persistence layer, a new permission axis, a new transaction scope is a human decision.
- **Security surface.** Any change that touches authentication, authorization, or secrets is reviewed by a human regardless of agent confidence.
- **Business policy.** Behavior that encodes "the business says X" is not inferred by the agent. It is written down and the agent implements it.
- **Backward compatibility.** Agents frequently propose "clean" changes that silently break older data shapes. Humans guard the compatibility contract.
- **"Should we even build this?"** The decision not to add a feature, or to delete one, is never delegated. Deletion in particular is only on explicit human request.

## 10. What the engineer's day looks like

The shift from "write code" to "define, dispatch, review, own":

- **Define** the task — write the requirement in a traceability file, mark the approvals explicitly.
- **Design** the solution with the agent — read what already exists, pick an approach, write the design file including the exact dispatch plan.
- **Dispatch** to specialized agents — implementation to a worker, review to a verifier pool, docs to the docs L1.
- **Review the deltas** — read the actual diffs, verify the test assertions, confirm principle compliance, trust nothing.
- **Own the result** — the PR has your name on it. The agent did not merge it. You did.

Less typing. Not less engineering.

## 11. What the model is good at

- Mechanical repetition at speed — find every call site, apply the same change, double-check, cross-reference.
- Reading large volumes of code faster than a human.
- Producing structured output on request (plans, findings, test cases).
- Catching trivial defects a human would miss on page 300 of a 500-page diff.

## 12. What the model is bad at

- Knowing when its model of the system is wrong. (This is why context is scoped and reviewers are adversarial.)
- Avoiding "clean refactors" that silently break something.
- Writing a test that would actually catch its own bug.
- Noticing architectural boundaries.
- Saying "I don't know". (Hence the plan-before-write UX on the in-product agent and the written-plan step on the build-time agent.)

## 13. Why this matters beyond this project

Enterprise software was already drifting toward unmaintainable per-customer forks before AI was in the picture. Unconstrained AI makes that worse at a faster rate.

What rescues the model is being specific about **what AI gets to decide vs. what humans still own**, and encoding the difference in a system that makes the right thing cheap and the wrong thing expensive. That is what this repository is about.
