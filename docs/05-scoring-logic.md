# Scoring logic

This note defines how *Responsible AI Evaluator* turns **use-case metadata** and **evaluation facts** into structured **risk** and **readiness** signals. It sits downstream of the core entities in [04-data-model.md](04-data-model.md) and upstream of human workflow and `governance_decisions`.

## Why a scoring layer exists

Raw tables are correct but not always **actionable** for review. The system needs a deterministic way to convert metadata and measured metrics into **structured signals** that support:

- **Review** — what to look at first, and why  
- **Prioritization** — which use cases or versions warrant deeper scrutiny  
- **Governance decisions** — evidence-shaped inputs, not a substitute for approvers  
- **Follow-up actions** — e.g. enhanced review, segment deep-dive, reapproval after drift  

The goal is **not** to automate ethics, replace policy, or collapse judgment into a number. It is to make **gaps and controls explicit** so reviewers spend time on the right questions.

## Design principles

- **Multi-dimensional over single-score** — outputs are **sets of indicators** and **required controls**, not one “ethics score.”  
- **Predictions remain separate from decisions** — scoring consumes **facts**; it does not **approve** deployment (see [Why this stays separate from governance decisions](#why-this-stays-separate-from-governance-decisions)).  
- **Explicit signals beat hidden heuristics** — rules and thresholds are **documented** and **versionable** like any other platform contract.  
- **Deterministic first, learned later** — v0 favors SQL / Spark / Python rules over opaque models.  
- **Reviewability over sophistication in v0** — a simple, inspectable rule set beats a clever black box for a portfolio and for audit.

## Inputs to scoring

Scoring jobs read **joined** inputs from the core model:

| Source | Role |
|--------|------|
| `ai_use_cases` | Context: `automation_level`, `human_in_loop`, `data_sensitivity`, `societal_impact_level`, `known_risk_flags`, geography and population scope |
| `model_versions` | Lifecycle: `model_stage`, MLflow linkage when present |
| `evaluation_runs` | **Which** evaluation (`run_id`, `evaluation_type`, `evaluation_timestamp`)—e.g. offline_holdout vs **post_deploy_drift** |
| `evaluation_metrics` | **What** was measured: `metric_name`, `metric_value`, `segment` |

Examples of facts that rules consume:

- **Automation and oversight** — `automation_level`, `human_in_loop`  
- **Sensitivity and impact** — `data_sensitivity`, `societal_impact_level`  
- **Segment behavior** — same `metric_name` across `segment` vs `overall` (e.g. false negative rate gap)  
- **Operational triggers** — `evaluation_type = post_deploy_drift` with materially worse error metrics vs baseline run  

Segment gaps and drift comparisons are **implemented** as joins and aggregates over `evaluation_metrics` keyed by `run_id` (and optionally compared across two runs for the same `model_version_id`).

## Output signals

Results are persisted as rows (e.g. in a **`scoring_results`** Delta table) with at least:

| Field | Meaning |
|-------|---------|
| `scoring_result_id` | Surrogate primary key for this scoring output |
| `use_case_id` | Scope of the use case |
| `model_version_id` | Artifact being scored |
| `run_id` | Evaluation run the scoring run was based on (primary evidence run; baseline may be implied by rule or separate column in a fuller schema) |
| `risk_indicators` | Structured list or delimited set of **codes** (e.g. `no_human_oversight`, `potential_disparate_impact`) — not prose judgments |
| `readiness_level` | Qualitative bucket (see below) — **advisory** summary |
| `required_controls` | Structured list or delimited set of **control codes** (e.g. `enhanced_review`, `reapproval_or_restriction_review`) |
| `scoring_timestamp` | When this result was produced |
| `scoring_notes` | Short, optional machine- or operator-generated explanation (e.g. “segment gap on false_negative_rate exceeded threshold”) |

`risk_indicators` and `required_controls` should use a **controlled vocabulary** so dashboards and workflows do not parse free text.

## Example deterministic rules

Rules are **illustrative**; production would tune thresholds and add org-specific exceptions. All are implementable in SQL / Spark / Python over the tables above.

| Condition | Effect |
|-----------|--------|
| `automation_level = automated` AND `human_in_loop = none` | Add risk indicator: `no_human_oversight` |
| `data_sensitivity = high` | Add required control: `enhanced_review` |
| `societal_impact_level = high` AND large segment gap on an agreed error metric vs `overall` (e.g. \|segment_fnr − overall_fnr\| > τ) | Add risk indicator: `potential_disparate_impact` |
| `evaluation_type = post_deploy_drift` AND agreed error metrics worsen materially vs baseline run (e.g. > τ% relative increase) | Add required control: `reapproval_or_restriction_review` |
| `known_risk_flags` contains `appeal required` (or substring policy) AND `human_in_loop = none` | Add risk indicator: `appeal_path_unclear` AND required control: `define_override_workflow` |

Materiality thresholds **τ** belong in configuration (table or file), not buried in code literals, so they are reviewable.

## Readiness levels

`readiness_level` is a **small qualitative label** for routing and narrative—not a green light to deploy.

| Level | Meaning |
|-------|---------|
| `exploratory` | Insufficient evidence, early stage, or blocking gaps; not a candidate for production promotion as-is |
| `review_required` | Evidence exists but indicators or missing controls require explicit human review before any promotion |
| `conditionally_ready` | Could proceed only with named controls and scope limits; ties to `required_controls` |
| `ready_with_controls` | Evidence supports progression **if** listed controls and scope are in place; still not an auto-approval |

**Readiness is not a deployment decision.** It is a **structured summary** for humans. Promotion remains a **`governance_decisions`** row with `approver_role`, `approved_scope`, and `rationale`.

## Why this stays separate from governance decisions

- **Scoring results** are **advisory system outputs** — derived from data and rules at a point in time.  
- **Governance decisions** are **organizational commitments** — who approved what, under which constraints, with accountability.  

A rule engine may **flag** risk or **require** controls; it does **not** approve deployment. That separation matches the repo’s core principle: **“Predictions are not decisions.”** Scores and indicators are closer to **predictions** (in the broad sense of inferred signals from data); **approval** is a **decision** recorded separately.

## Example walkthrough (hiring screening)

**Context:** `ai_use_cases` row for resume screening — `automation_level = semi_automated`, `human_in_loop = override`, `data_sensitivity = high`, `societal_impact_level = high`.  

**Evidence:** `evaluation_runs` = offline_holdout for `model_version_id` v1; `evaluation_metrics` show `false_negative_rate` materially higher for `segment = source_channel_referral` than for `segment = overall`.  

**Rule firings:**  
- `data_sensitivity = high` → `required_controls` includes `enhanced_review`  
- high impact + segment gap → `risk_indicators` includes `potential_disparate_impact`  
- semi-automated + override present → do **not** add `no_human_oversight`  

**Readiness:** `conditionally_ready` — contingent on completing enhanced review and documenting mitigation or acceptance of segment risk in `governance_decisions.rationale`.  

A product or policy lead still **decides**; scoring only **queues** the right questions and controls.

## Implementation note

v0 should be **rule-based** (Spark or SQL batch job, or Python with clear rule modules), writing rows to **`scoring_results`** in Delta. Rule packs can be versioned (e.g. `scoring_rule_version` column) so historical rows remain interpretable.

If **learned** components are introduced later, they should remain **explainable and reviewable** (e.g. feature attributions, constrained model cards, or human-in-the-loop override of model-generated ranks)—and must **not** replace `governance_decisions` or imply single-score “ethics.”
