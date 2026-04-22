# Problem statement: why “good models” are not enough

## The gap

Many companies run mature **delivery** practices for software and increasingly for ML: CI/CD, registries, canaries, and dashboards. They are weaker at **evaluating** AI initiatives as **initiatives**—embedded in products, org incentives, and external effects.

The usual build pipeline answers **“can we build it?”** (quality on a test set, latency, cost). The harder questions are **“should we build or deploy it?”** and **“for whom, under what constraints, and with what recourse if it is wrong?”** Those questions cut across data fit, product risk, **governance**, **human oversight**, and **societal impact**. When they are unowned or answered only in static documents, you get a persistent gap: **the system in production** does not match **the story** told at launch.

## “Can we build it?” vs “Should we build it?”

| Lens | “Can we build it?” (common default) | “Should we build it?” (this project’s emphasis) |
|------|--------------------------------------|-----------------------------------------------|
| Evidence | Leaderboard metrics, A/B lift | Fit to purpose, error costs by segment, misuse patterns |
| Time horizon | Release window | Drift, policy change, incident history |
| Success owner | Data science / eng metrics | **Accountable** product + **policy** + **operations** |
| Failure | Model error in aggregate | **Who gets hurt**, procedural fairness, compounding in workflows |

None of that replaces rigorous ML engineering. It **sits on top of it**—the way SLOs sit on top of code: without explicit ownership and telemetry, you do not know whether you are “good enough” in the real world.

## Governance: not paperwork, a control plane

**Governance** here means **decision rights and evidence**: which roles can approve deployment, on what **basis**, and with what **record**. If governance lives only in meetings, the production system is ungoverned.

A serious approach keeps **governance next to the artifacts**:

- **What** was evaluated (metrics, populations, time window)  
- **Which** model version and **which** data snapshot  
- **Who** signed off, **when**, and for **which scope** (one market vs another, batch vs online)  
- **What** must happen on breach (rollback, retrain trigger, human queue)

## Human oversight: where judgment actually lives

Oversight is not “a human looked at a dashboard once.” It is **defined workflow**: override authority, **reason codes**, and **audit** when automation is wrong. Platform engineers recognize this pattern—it is the same as **on-call** and **incident command**: clarity of roles under stress.

**Responsible AI Evaluator** frames oversight as part of the **data model** and **process**, not a generic checkbox. Predictions and decisions remain **distinguishable** in storage and in workflow.

## Societal impact: grounded, not performative

Societal impact is not only media-facing ethics. In practice it shows up as:

- **Allocation effects** (who is included or excluded by thresholds and workflows)  
- **Feedback loops** (model outputs that change future data or user behavior)  
- **Concentration of error** (segments that are under-represented in training or in monitoring)  

This project does not solve social outcomes. It **structurally allows** the team to **record** known segment risks, **monitor** for drift on those dimensions where appropriate, and **tie** post-incident review back to the same **evidence chain** you would use in a well-run reliability program.

## Why a system (and not a memo)

Memos and model cards are necessary and insufficient. As in any platform, **if it is not in the data path, it will drift**. A small, honest system that links **MLflow + Delta + review records** is a **concrete** answer to: *how would we show our work in an audit, a postmortem, or a product review?*

That is the problem this repository is designed to **address in scope**—and to **refuse to fake** out of scope.
