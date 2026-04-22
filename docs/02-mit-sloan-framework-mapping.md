# MIT Sloan framework mapping → system design

This document maps ideas common in **management-of-AI** education (e.g. MIT Sloan–style programs on strategy and responsible use of AI) to **concrete** choices in *Responsible AI Evaluator*. It is a **design bridge**, not a course summary.

**Legend:** *Concept* → *System implication* → *How this project implements it* (or will, per roadmap)

---

## 1. Predictions vs decisions

| | |
|--|--|
| **Concept** | Models **predict**; **people, policy, and governed automation** **decide**. Conflating the two is how accountability disappears. |
| **System implication** | Separate **prediction records** (scores, class, uncertainty) from **decision records** (approve, route, block, set threshold) in schema and in workflow. |
| **In this project** | Evaluation tables hold **model outputs and metrics**; a distinct **governance / approval** store (or table family) records **actor, time, scope, and rationale**. No field named “decision” should silently copy `prediction` without an explicit step. The charter encodes: **“Predictions are not decisions.”** |

---

## 2. Human + AI collaboration

| | |
|--|--|
| **Concept** | Value often comes from **pairing** human judgment with model throughput—not from replacing humans in ill-defined “steps.” |
| **System implication** | Systems need **roles**, **queues**, and **handoff** metadata—not a single “AI score” as the only UI. |
| **In this project** | **Roadmap** includes review queues, **reason codes** for override, and linkage from **MLflow run / model version** to the **case** a human saw. Optional notebooks or a thin UI are **read/write boundaries** for those roles, not a demo net. |

---

## 3. Augmentation vs automation

| | |
|--|--|
| **Concept** | **Augmentation** helps a human decide; **automation** commits an action. The latter demands stronger controls, monitoring, and explicit scope. |
| **System implication** | Configuration must record **automation level** and **allowed actions**; monitoring and rollback must match that level. |
| **In this project** | Metadata fields (and later policy checks) tag whether a use case is **advisory** vs **automated**; **automated** paths require stricter **eval completeness** and **incident** hooks in design. Spark/Delta recompute jobs support “**we re-evaluated after drift**” as an operational fact. |

---

## 4. Operator / workflow perspective

| | |
|--|--|
| **Concept** | AI systems fail in **process**, not only in **math**: bad inputs, wrong cohort, wrong threshold in the wrong country, stale registry entry. |
| **System implication** | Treat “operator” as a first-class concern: idempotent jobs, **versioned** inputs, runbooks, clear ownership between DS and platform. |
| **In this project** | Framed in **reliability** terms: **batch eval** pipelines with explicit **batch IDs**, **Delta time travel** for “what did we use that day?”, and MLflow for **which artifact** ran. This mirrors how platform teams treat **deployments and rollbacks**—here applied to **eval and approval state**. |

---

## 5. Organizational impact

| | |
|--|--|
| **Concept** | AI changes **incentives, headcount, and power** in workflows; “model accuracy” does not measure that. |
| **System implication** | Capture **process change**: who is removed from the loop, who gains veto power, what new failure modes exist for customer support. |
| **In this project** | **Non-goals** and charter avoid fake quantification. The system **reserves** fields / docs for **operating model** notes (RACI-style summaries) tied to a **model version and scope**—enough to show **informed** portfolio thinking without claiming HR analytics. |

---

## 6. Societal impact

| | |
|--|--|
| **Concept** | Externalities—allocation, feedback loops, and disparate harm—are **operational** questions, not a slide on “ethics.” |
| **System implication** | **Segmented** evaluation, **known limitations**, and **monitoring** on vulnerable cohorts (where data and law allow) must be part of the **data layer**, not an appendix. |
| **In this project** | **Slice / segment** metrics in Spark-driven tables; **documented** known blind spots; **drift** and **re-eval** in the roadmap as *platform* concerns—same as capacity or error budgets in mature services. |

---

## 7. Governance

| | |
|--|--|
| **Concept** | **Governance** is **accountability and evidence**, not a bolt-on. It must be **enforced** where artifacts move—from experiment to registry to production. |
| **System implication** | **Immutable-ish** audit log, **role-bound** actions, and **version** discipline across data and models. |
| **In this project** | **MLflow** for experiment and registry lineage; **Delta** for **ACID** and **history**; **explicit approval** rows keyed to **model version + use-case scope**. **CI** (when added) treats **docs and schema** as part of the **contract**—the same habit as in platform engineering. |

---

## Cross-cutting note (platform + reliability)

A thread through all rows: **responsible** deployment looks like **good operations** at scale—**observable**, **reversible**, and **attributable**. This project is intentionally framed that way: not “ethics or engineering,” but **ethics *through* engineering** where the org already knows how to be serious (incidents, SLOs, change control).

---

## Document control

- **Status:** Living mapping for portfolio design  
- **When to update:** New lifecycle phase (e.g. online serving), new regulatory context, or material stack change
