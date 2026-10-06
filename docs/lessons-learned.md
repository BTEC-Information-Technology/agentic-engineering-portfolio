# Lessons learned

> Non-generic findings from running an AI-native development model across hundreds of real engineering sessions over the lifetime of the Platform. These are not LLM-advice generalities — each one is grounded in a specific class of failure or success observed on this project.

## 1. Specialization beats scale

A pool of narrow verifiers, each with a tailored prompt and tool allow-list, catches more real defects than one large generalist agent reviewing the whole diff. The reason is not raw capability — it is *stake*. A generalist agent reviewing its own class of code is biased toward confirming its own patterns. A narrow verifier reviewing in a lane it did not write has no such bias.

The practical form of this: six L1 roles (docs, front-end, back-end, tester, DB, communications) rather than "the reviewer". Each has a different question list. Each produces a different kind of finding.

## 2. Context size has a cliff, not a slope

Beyond a certain context fill, agents stop reading and start pattern-matching. The output starts looking right against the shape of the context, not right against the specific file. The failure mode is quiet: plausible wrong code at full confidence.

The counterintuitive consequence: **more context is not more correct**. Deliberately scoping the context — the files the task actually touches, the principles document, the design file, and nothing else — is a correctness mechanism, not a cost mechanism. "The agent has seen it all" is a sign the agent is about to make an unforced error.

## 3. The dangerous output is the plausible one

Obviously wrong plans are easy to reject. Plans that look right against a flawed model of the system are the ones that ship bugs. These are the hard reviews, because the reviewer must notice a subtle mismatch between the plan's model and the real system — not a surface defect.

Three mitigations that work:

- **Plan-before-write.** The agent writes the plan down before it writes any code, and the plan is reviewed first. A plausible-wrong plan is cheaper to catch than plausible-wrong code.
- **Adversarial review.** The reviewer's task is "argue this is wrong", not "verify this is right". Different question, different output.
- **File-and-line references in findings.** A finding without a specific file and line is treated as unverified and dropped.

## 4. Mocks in AI-written tests are a trap

An agent asked to "write tests" will produce tests that pass. Of course it will — it is also the thing writing the code those tests check. If both sides share a flawed model, the tests validate the flaw.

The move: tests that touch persistence must run against a real database. The assertions must be reviewed by a human before merge. "100% coverage" is a measure, not evidence.

A corollary: a green CI build is not enough. The engineer owning the PR reads the assertions, not just the pass/fail.

## 5. Hallucinations are structured, not random

LLM hallucinations have a shape. On an enterprise codebase the common ones are:

- confusing two caches with similar names
- confusing two schemas with similar names
- recommending a flag or a function that was renamed or removed
- believing a config value is read where it is not (because a getter exists for the key but is unused)

These are not "random mistakes". They are predictable classes of error. Preventing them is a matter of guardrails per class:

- **Confusing caches:** the architecture document explicitly names each cache, its scope, and its invalidation lifetime.
- **Confusing schemas:** schema resolution is defined once in a helper, and the helper is the only place the mapping lives.
- **Phantom flags:** before recommending a flag, grep for it. Memory is a snapshot; the code moved.
- **Phantom config reads:** before recommending a config change, trace the getter to the first consumer.

A reviewer that knows the shape of hallucinations catches them faster.

## 6. The verbatim-approval rule prevents silent scope drift

Agents paraphrase. The paraphrase quietly turns "yes, in this one call site" into "yes, across the file". The fix: copy the user's actual words into the design file. Not a summary.

This is one of the lowest-cost, highest-value rules on the project. It takes thirty seconds per approval and prevents the single most expensive review failure.

## 7. The CI reviewer is the forcing function

Local hooks catch most of the mechanical defects. The CI reviewer catches the rest. It exists because any local hook can be `--no-verify`-d, and sometimes is.

The right posture for the CI reviewer is fail-closed: the PR cannot merge until it passes. Not advisory. Not "the human can override". A gate that can be routed around becomes a gate that is routed around.

## 8. Memory per developer is the right unit

A shared "project memory" is useful. A per-developer memory is more useful. The reason is that preferences and corrections are personal — one engineer wants terse responses, another wants verbose; one wants auto-commit on modifications, another wants a confirmation. A single shared memory would be either too narrow (not enough coverage of real preferences) or too noisy (every engineer's corrections mixed together).

Per-developer memory, loaded automatically at every session, kept small, versioned by the engineer themselves. The shared layer is for the project — the architecture, the principles, the gotchas.

## 9. Translation discipline is a maturity signal

A codebase that routes every user-visible string through a translation layer from day one scales to more languages, more customers, and more teams without any new mechanism. A codebase that bolts on translations after the fact has string-literal archaeology to do for months.

The rule "every user-visible string is a translation key" is cheap at the start, expensive to add later. It is also one of the places where agents regress most often — a hard-coded error message slips in and the fix is a sweep. A hook that blocks string literals in known-visible places pays for itself quickly.

## 10. Live mirroring beats "deploy to test later"

On the reference environment, every masterdata edit replicates to the registered test deployments in the same transaction. The alternative — "I made changes on the reference, I will push to test tomorrow" — is where most state divergence incidents start.

Live mirror is a design choice that sharply reduces an entire class of synchronization bugs — it collapses the "I made the change here but not there" window to zero. It also makes "it works on my box" harder, because the mirror is the proof.

## 11. Specific things never to delegate

A short list that has earned its place:

- choosing to **remove** code
- changes to **authorization**
- changes to the **capture pipeline** or any trigger
- changes to the **secrets vault configuration**
- running **destructive** scripts on reference data
- opening a **pull request**
- any `--force` on git

Agents can propose any of these. They cannot do any of these without an explicit human `go`.

## 12. The engineering job did not shrink

Writing less code does not mean less engineering. The attention has moved from "write" to "define, dispatch, review, own". The hard parts — understanding the system, maintaining its invariants, deciding what belongs in the engine vs. what does not, protecting backward compatibility, aligning with the business — are still fully human.

An engineer who treats AI as "the thing that writes the code so I can skip to the next task" produces lower-quality output than one who treats AI as "the thing that handles the mechanical part so I can spend more time on the hard part".

## 13. Documentation is a first-class artifact

The in-product docs feed the end-user assistant. The architecture document guides the agents. The principles document is the behavioral contract. The lessons-learned document (this one) prevents the team from re-paying for mistakes it already made.

A document that nobody reads is a cost. A document that every session reads is leverage. The build-time agents read these documents at session start; the in-product end-user assistant retrieves from them. Writing them well pays twice.

## 14. The thing that surprised me most

Early on, the assumption was that the hard problem would be the model — picking the right one, keeping cost down, latency up. In practice, the hard problem has been the system around the model: scoping context, dispatching correctly, enforcing principles, preserving human ownership, keeping artifacts truthful.

The model is a component. The *system* is what makes AI engineering actually work.
