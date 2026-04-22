# Data model

This note defines the **core relational shape** of *Responsible AI Evaluator*: enough structure to implement in Delta (or any SQL store) without pretending to be a full enterprise MRM system.

## Design goals

The data model must:

- **Preserve traceability** — any governance record can be tied to a specific model version, evaluation run, and time window.  
- **Separate predictions from decisions** — model outputs and human or policy **approvals** are different facts, stored in different places.  
- **Support audit and review** — approvers, scope, and rationale are first-class fields, not prose-only.  
- **Stay small** — appropriate for a portfolio: a handful of core tables, explicit keys, minimal cleverness.

## Core entities

### `ai_use_cases`

| | |
|--|--|
| **Purpose** | The **unit of product and policy context**: what the system is for, who it affects, and how automated it is. |
| **Key fields** | `use_case_id` (PK), `use_case_name`, `domain`, `automation_level` (e.g. advisory / semi_automated / automated), `human_in_loop` (boolean or enum: none / review / override), optional notes for deployment context and risk flags. |
| **Why it exists** | Evaluations and approvals are meaningless without **scope**—same model can be appropriate in one use case and not in another. This table anchors that scope. |

### `model_versions`

| | |
|--|--|
| **Purpose** | A **versioned** trainable artifact (or registered model) tied to ML lifecycle metadata. |
| **Key fields** | `model_version_id` (PK), `use_case_id` (FK), `model_name`, `model_stage` (e.g. development / staging / production), optional `mlflow_experiment_id`, `mlflow_run_id`, `mlflow_model_version` for linkage. |
| **Why it exists** | “Which artifact?” must be a stable key for eval batches and for governance—**lineage** from experiment to decision. |

### `evaluation_runs`

| | |
|--|--|
| **Purpose** | One **batch** of evaluation (offline holdout, backtest, red-team set, or scheduled drift check) against a **fixed** `model_version_id`. |
| **Key fields** | `run_id` (PK), `model_version_id` (FK), `evaluation_type` (e.g. offline_holdout, slice_audit, post_deploy_drift), `evaluation_timestamp`, optional `data_snapshot_id` or dataset URI for reproducibility. |
| **Why it exists** | Metrics are not global—they belong to a **run** at a **time** with a defined **data** boundary. Re-eval after drift = new run, same or new model version. |

### `evaluation_metrics`

| | |
|--|--|
| **Purpose** | Atomic **measured** facts: one metric value, optionally for a **segment**. |
| **Key fields** | `run_id` (FK), `metric_name` (e.g. false_negative_rate, auc), `metric_value`, `segment` (e.g. `overall` or a slice key), optional `metric_unit` or `details_json`. |
| **Why it exists** | Slices and overall stats are many rows, not a wide sprawl of columns—**Spark-friendly**, easy to add metrics without schema thrash. |

### `governance_decisions`

| | |
|--|--|
| **Purpose** | A **decision** to allow, change, or stop something—**not** a model output. |
| **Key fields** | `decision_id` (PK), `use_case_id` (FK), `model_version_id` (FK), `approver_role`, `decision_outcome` (e.g. approved / approved_with_conditions / rejected / deferred), `approved_scope` (what is allowed: geography, channel, max automation), `rationale` (short text), `decision_timestamp`. |
| **Why it exists** | Implements the charter: **someone** commits **under constraints**; that record is queryable and tied to **evidence** (evaluation runs) by foreign keys and process, not by embedding scores into “decision.” |

## Entity diagram (text)

```text
ai_use_cases
    ↓
model_versions
    ↓
evaluation_runs
    ↓
evaluation_metrics

governance_decisions
    ↘ references use_case + model_version
```

## Entity relationships

- One **use case** has many **model versions** over time (retrains, experiments).  
- One **model version** has many **evaluation runs** (initial sign-off, periodic drift, post-incident recheck).  
- One **evaluation run** has many **evaluation metrics** (overall + segment rows).  
- **Governance decisions** reference a **specific** `model_version_id` and **use case** scope: approval is for “this version, this use case, these constraints.”  

Join path for a typical audit: `governance_decisions` → `model_version_id` → `evaluation_runs` → `evaluation_metrics`, all scoped by `use_case_id` where needed.

## Why predictions and decisions are separate

**Model outputs** (scores, classes, rank order) are produced by a training recipe and an input; **governance decisions** are commitments by an organization—who may require **new** information (legal, PR, product) that the model never sees.

If those are stored as one record:

- you lose **who** decided and **on what basis**;  
- you invite **quiet overwrite** of “decision” when the model is re-scored;  
- you conflate the MIT Sloan distinction this repo maps to the schema: [predictions vs decisions](02-mit-sloan-framework-mapping.md), aligned with the charter principle **“Predictions are not decisions.”**  

So: **facts about the model** live in `evaluation_runs` / `evaluation_metrics` (and, at inference time, in application logs—out of scope for this core model unless extended). **Facts about approval** live in `governance_decisions`.

A governance decision may reference a specific evaluation run directly, or through a stricter linking pattern in a fuller implementation. In the portfolio version, the minimum viable pattern is a foreign key to `model_version_id` plus a timestamp that allows the relevant evaluation runs in force at approval time to be reconstructed.

## Example lifecycle (hiring screening)

1. **Register** `ai_use_cases`: e.g. `use_case_id = hiring_resume_screen_eu`, `automation_level = semi_automated`, `human_in_loop = override`.  
2. **Register** `model_versions`: v1 of the resume ranker, `model_stage = staging`, MLflow IDs filled when wired.  
3. **Run** `evaluation_runs`: `evaluation_type = offline_holdout`, `evaluation_timestamp` set, `data_snapshot_id` for the label set.  
4. **Insert** `evaluation_metrics`: `false_negative_rate` for `segment = overall` and for `segment = region_emea` (or protected-class slices where policy allows measurement).  
5. **Record** `governance_decisions`: `decision_outcome = approved_with_conditions`, `approved_scope = EU_only; max_auto_shortlist_30pct`, `rationale` referencing false rejection concerns in specific segments, `decision_timestamp` before production promotion.  
6. **Later**, a drift job creates a **new** `evaluation_runs` row; new metrics can be compared to the run(s) in force at the original `decision_timestamp` without rewriting history.

## Practical constraints

In a real system, not all segments or sensitive attributes can be measured or stored due to legal, policy, and data availability constraints. This design assumes that evaluation and segmentation operate within those boundaries and that some risks may only be partially observable.

## Implementation note

A sensible portfolio order is: implement these entities as **Delta** tables (and views) with explicit constraints and partitioning by `use_case_id` or time as volume grows. **MLflow** linkage on `model_versions` can be added **incrementally** so v1 of the repo stays honest: schema and batch eval first, registry IDs second.
