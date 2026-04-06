---
name: metaopt-experiment-design
description: "Use when the ml-metaoptimization orchestrator needs to design an experiment from the winning proposal. Produces a concrete batch specification including code change guidance, search space, and execution assumptions. Keywords: experiment design, batch specification, execution plan, metaoptimization worker."
---

# metaopt-experiment-design

## Overview

Transform the winning proposal into a concrete experiment batch specification suitable for the backend contract.
This is the bridge between a high-level idea and an actionable execution plan — it takes the selected proposal and produces a design detailed enough for the materialization worker to implement without ambiguity.

This skill is a leaf worker operating in the **design** auxiliary lane.
It receives the winning proposal and campaign context from the orchestrator via a subagent prompt, produces a concrete experiment specification, and returns it.
It does not generate code, interact with files, dispatch subagents, or manage state.

**Lane:** Auxiliary slot — `design`
**Model class:** `strong_reasoner` (prefer a strong reasoning model like Opus 4.6 fast, fallback to a capable general model like GPT-5.4)

## Input Contract

The orchestrator supplies all inputs as structured context in the subagent prompt. This skill never reads files directly from the campaign repo.

### Standard Envelope (provided by orchestrator on every dispatch)

| Field | Type | Description |
|-------|------|-------------|
| `campaign_id` | string | Campaign identifier |
| `current_iteration` | integer | Current iteration number |
| `slot_id` | string | The slot ID dispatching this worker |
| `attempt` | integer | Attempt number for this dispatch (1-indexed) |

### Winning Proposal

| Field | Type | Description |
|-------|------|-------------|
| `winning_proposal` | object | The complete proposal selected by the synthesis phase |
| `winning_proposal.proposal_id` | string | Non-empty identifier for the proposal |
| `winning_proposal.title` | string | Concise name for the experiment |
| `winning_proposal.rationale` | string | Why this change is expected to improve the metric |
| `winning_proposal.target_area` | string | Which pipeline aspect this targets |
| `winning_proposal.expected_impact` | object | `{ direction, magnitude }` — the ideation worker's impact assessment |

### Campaign Context

| Field | Type | Description |
|-------|------|-------------|
| `goal` | string | The campaign's top-level improvement goal |
| `metric` | string | The objective metric being optimized |
| `direction` | `"minimize"` or `"maximize"` | Whether lower or higher metric values are better |
| `aggregation_method` | string | How per-dataset scores roll up into the aggregate (e.g. `mean`, `weighted_mean`) |
| `aggregation_weights` | object or null | Per-dataset weights when method is `weighted_mean`; `null` otherwise |

### Dataset Definitions

| Field | Type | Description |
|-------|------|-------------|
| `datasets` | array | Dataset entries from the campaign spec |
| `datasets[].id` | string | Dataset identifier |
| `datasets[].role` | string | Dataset role (e.g. `train`, `eval`, `test`) |
| `datasets[].fingerprint` | string | Content-addressed fingerprint for the dataset |

### Execution Config

| Field | Type | Description |
|-------|------|-------------|
| `execution` | object | Execution configuration from the campaign spec |
| `execution.runner_type` | string | How trials are executed (e.g. `ray_tune`, `subprocess`) |
| `execution.entrypoint` | string | Shell command used to run the experiment |
| `execution.trial_budget` | object | `{ kind: string, value: number }` — budget type and amount |
| `execution.search_strategy` | object | `{ kind: string, ...params }` — search strategy and parameters |

### Baselines

| Field | Type | Description |
|-------|------|-------------|
| `aggregate_baseline` | number | Current aggregate baseline score |
| `per_dataset_baselines` | object | Map of dataset IDs to their current numeric baseline values |

### Historical Context

| Field | Type | Description |
|-------|------|-------------|
| `key_learnings` | array | Learnings extracted from prior iterations |
| `completed_experiments` | array | Summary of all previously run experiments and their outcomes |

### Backend Contract Requirements

| Field | Type | Description |
|-------|------|-------------|
| `backend_contract` | object/string | Requirements from `ml-metaoptimization/references/backend-contract.md` describing what the backend expects for enqueue, status, and results |

## Output Contract

Return a response containing the following sections.

### Experiment Specification (required)

| Field | Type | Description |
|-------|------|-------------|
| `experiment_name` | string | A descriptive name for the experiment batch (≤ 15 words) |
| `experiment_description` | string | One-paragraph summary of what the experiment does and why |
| `proposal_id` | string | Echoes the winning proposal's `proposal_id` |

### Code Change Guidance (required)

| Field | Type | Description |
|-------|------|-------------|
| `code_changes` | array | File-level guidance for the materialization worker |
| `code_changes[].file_path` | string | Relative path to the file that needs modification |
| `code_changes[].change_type` | string | One of `modify`, `create`, `delete` |
| `code_changes[].description` | string | What the change should accomplish |
| `code_changes[].key_constraints` | array of strings | Important constraints the implementation must respect |

This is a specification, not code. Each entry describes *what* must change and *why*, not *how* to write the code.

### Hyperparameter Search Space (required when applicable)

| Field | Type | Description |
|-------|------|-------------|
| `search_space` | object or `null` | Hyperparameter search space definition; `null` if no hyperparameter search is involved |
| `search_space.parameters` | array | Parameters to search over |
| `search_space.parameters[].name` | string | Parameter name |
| `search_space.parameters[].type` | string | One of `continuous`, `discrete`, `categorical` |
| `search_space.parameters[].range` | array or object | Value range or set of allowed values |
| `search_space.parameters[].default` | any | Default or baseline value if known |
| `search_space.strategy` | string | Must match `execution.search_strategy.kind` |
| `search_space.trial_budget` | integer | Must not exceed `execution.trial_budget.value` |

### Dataset Usage Plan (required)

| Field | Type | Description |
|-------|------|-------------|
| `dataset_plan` | array | How each dataset is used in the experiment |
| `dataset_plan[].dataset_id` | string | Matches an entry from the campaign's `datasets` |
| `dataset_plan[].role` | string | Role in this experiment (e.g. `train`, `eval`, `holdout`) |
| `dataset_plan[].notes` | string | Any special handling or preprocessing required |

### Artifact Expectations (required)

| Field | Type | Description |
|-------|------|-------------|
| `artifact_expectations` | object | What the materialization phase must produce |
| `artifact_expectations.code_artifact` | string | Description of the immutable code artifact expected |
| `artifact_expectations.data_manifest` | string | Description of the data manifest expected |
| `artifact_expectations.additional_artifacts` | array of strings | Any other artifacts the experiment requires |

### Success Criteria (required)

| Field | Type | Description |
|-------|------|-------------|
| `success_criteria` | object | How to judge whether the experiment succeeded |
| `success_criteria.metric` | string | Must match the campaign's `metric` |
| `success_criteria.direction` | string | Must match the campaign's `direction` |
| `success_criteria.minimum_improvement` | number or `null` | Minimum delta over aggregate baseline to count as success; `null` if any directional improvement suffices |
| `success_criteria.per_dataset_expectations` | object or `null` | Optional expected per-dataset outcomes |

### Execution Assumptions (required)

| Field | Type | Description |
|-------|------|-------------|
| `execution_assumptions` | object | Resource and runtime expectations |
| `execution_assumptions.runner_type` | string | Must match the campaign's `execution.runner_type` |
| `execution_assumptions.estimated_duration` | string | Human-readable estimate (e.g. `"2-4 hours"`) |
| `execution_assumptions.resource_requirements` | string | Expected compute resources (e.g. `"1x GPU per trial"`) |
| `execution_assumptions.parallelism` | string | Expected trial parallelism (e.g. `"up to 4 concurrent trials"`) |

### Risks and Caveats (required)

| Field | Type | Description |
|-------|------|-------------|
| `risks` | array | Known risks, assumptions, or caveats the orchestrator should track |
| `risks[].description` | string | What the risk is |
| `risks[].mitigation` | string | How to mitigate or what to watch for |

## Behavioral Rules

1. **Design, don't code.** Produce a specification that guides code changes. Never generate code, patches, diffs, or implementation artifacts. The materialization worker is responsible for implementation.

2. **Concrete enough to implement.** The design must be specific enough that the materialization worker can produce the changeset without ambiguity. Vague guidance like "improve the model" is insufficient — specify which files change, what the changes achieve, and what constraints apply.

3. **Respect the trial budget.** The `search_space.trial_budget` must not exceed `execution.trial_budget.value` from the campaign config. If the proposal implies more trials than allowed, constrain the search space to fit within budget.

4. **Match the search strategy.** The `search_space.strategy` must match the campaign's `execution.search_strategy.kind`. Do not invent a different search approach.

5. **Match the runner type.** The `execution_assumptions.runner_type` must match the campaign's `execution.runner_type`. Design the experiment to be compatible with the declared execution environment.

6. **Use all relevant datasets.** The `dataset_plan` must reference datasets from the campaign's `datasets` array. Do not invent datasets that don't exist. Every dataset with role `eval` must appear in the plan.

7. **Reference baselines explicitly.** Success criteria must be stated relative to the current aggregate baseline and, when relevant, per-dataset baselines. Do not define success in absolute terms without baseline context.

8. **Leverage prior learnings.** Use `key_learnings` and `completed_experiments` to inform the design. If a prior experiment already explored a similar search space region, narrow the space or shift the focus accordingly.

9. **Backend compatibility.** The design must produce artifacts compatible with the backend contract. The materialization worker needs to produce an immutable code artifact and data manifest that the backend can consume via the enqueue contract.

10. **No state mutation.** This skill does not read or write state files, campaign files, or any persistent artifacts. It receives context and returns a design.

11. **Echo the proposal ID.** The output must include the `proposal_id` from the winning proposal so the orchestrator can track lineage.

12. **Flag risks honestly.** If the proposal involves assumptions that might not hold, non-obvious failure modes, or areas where prior learnings suggest caution, list them in `risks`. Do not suppress caveats to appear confident.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing actual code instead of a specification | Describe what changes are needed at the file level, not how to write the code |
| Exceeding the trial budget with the search space | Check `execution.trial_budget.value` and constrain `search_space.trial_budget` accordingly |
| Using a search strategy different from the campaign config | Match `search_space.strategy` to `execution.search_strategy.kind` exactly |
| Leaving code change guidance too vague to implement | Specify file paths, change types, and what each change must accomplish |
| Ignoring per-dataset baselines when defining success | Reference both aggregate and per-dataset baselines in `success_criteria` |
| Omitting the dataset usage plan | Every dataset in the campaign must be accounted for in `dataset_plan` |
| Designing artifacts incompatible with the backend contract | Ensure the design expects immutable code artifacts and data manifests, not mutable working tree paths |
| Repeating a search space region already explored | Check `completed_experiments` and `key_learnings` to avoid redundant exploration |
| Providing no risk assessment | Always include at least one risk or caveat, even for straightforward experiments |
| Forgetting to echo the proposal ID | Include `proposal_id` in the experiment specification for lineage tracking |
| Defining success without referencing the baseline | State minimum improvement relative to `aggregate_baseline`, not in absolute terms |
| Designing for a runner type different from the campaign config | Match `execution_assumptions.runner_type` to `execution.runner_type` |

## References

This skill is part of the `ml-metaoptimization` orchestrator ecosystem:

- **`ml-metaoptimization/references/worker-lanes.md`** — Authoritative lane contract for the design slot, including input/output expectations and slot class rules.
- **`ml-metaoptimization/references/contracts.md`** — State file schema, batch manifest contract, selected experiment contract, and slot field definitions.
- **`ml-metaoptimization/references/backend-contract.md`** — What the backend expects for enqueue, status, and results — the design must produce artifacts compatible with this contract.
- **`ml-metaoptimization/SKILL.md`** — Orchestrator skill contract defining dispatch invariants and worker policy.
