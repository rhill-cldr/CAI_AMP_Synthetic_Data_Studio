# Evaluation Pipeline

Source: `app/services/evaluator_service.py`, `app/services/evaluator_legacy_service.py`

## Overview

Two evaluator services implement LLM-as-judge scoring. Both produce a numeric score and a text justification for each item in the input dataset.

| Service | Technique | Entry Point | Input Unit |
|---|---|---|---|
| `EvaluatorService` | Freeform | `evaluate_row_data()` | Arbitrary dict (row) |
| `EvaluatorLegacyService` | SFT, Custom\_Workflow | `evaluate_results()` | QA pair keyed by `output_key` / `output_value` |

Both services share the same structural pattern: create a model handler, load data from a JSON file, fan out evaluations in parallel via `ThreadPoolExecutor`, collect scores and failures, compute summary statistics, persist results and metadata.

---

## Legacy Evaluation (EvaluatorLegacyService)

**Entry point:** `evaluate_results(request, job_name, is_demo)`

### Pipeline Steps

1. Create a model handler via `create_handler()` using parameters from the request (`model_id`, `inference_type`, `caii_endpoint`, `model_params`).
2. Load data from `request.import_path` (a JSON file containing a list of items).
3. Transform data: group items by the `Seeds` topic key into a `transformed_data["results"]` dict, extracting `output_key` and `output_value` fields into QA pair dicts.
4. Submit each topic to `evaluate_topic()` via a top-level `ThreadPoolExecutor`.
5. Within `evaluate_topic()`, each QA pair is submitted to `evaluate_single_pair()` using a nested `ThreadPoolExecutor`.
6. `evaluate_single_pair()` validates that both `output_key` and `output_value` exist in the pair, then builds the prompt via `PromptBuilder.build_eval_prompt()`.
7. The model response is parsed for `score` and `justification` fields.
8. Default error response on failure: `score=0`, `justification="Error during evaluation"`.
9. Per-topic and overall statistics (average, min, max) are calculated from successful evaluations only.
10. Results are saved to `qa_pairs_{model}_{timestamp}_evaluated.json`.

### Validation

Each QA pair must contain both the `output_key` and `output_value` fields specified on the request. Missing keys produce an error response without calling the LLM.

---

## Freeform Evaluation (EvaluatorService)

**Entry point:** `evaluate_row_data(request, job_name, is_demo)`

### Pipeline Steps

1. Create a model handler via `create_handler()` using request parameters.
2. Load data from `request.import_path`. If the loaded JSON is not a list, it is wrapped in one.
3. Call `evaluate_rows()` which submits all rows to `evaluate_single_row()` in parallel.
4. `evaluate_single_row()` builds the prompt via `PromptBuilder.build_freeform_eval_prompt()`, passing the entire row dict (not individual fields).
5. The model response is parsed for `score` and `justification` fields.
6. Default error response on failure: `score=0`, `justification="Error during evaluation"`.
7. Summary statistics (average, min, max) are calculated from successful evaluations only.
8. An `Overall_Average` key is appended to the top-level results dict.
9. Results are saved to `row_data_{model}_{timestamp}_evaluated.json`.

---

## Concurrency

Both services use `ThreadPoolExecutor` with `functools.partial` for clean argument binding and `concurrent.futures.as_completed()` to process results as they finish.

| Parameter | `EvaluatorService` Default | `EvaluatorLegacyService` Default | Override |
|---|---|---|---|
| `max_workers` | 5 | 4 | `request.max_workers` (takes precedence when set) |

The legacy service creates two levels of parallelism: topics are evaluated in parallel at the top level, and within each topic, individual QA pairs are evaluated in parallel.

The freeform service uses a single level: all rows are evaluated in parallel within `evaluate_rows()`.

---

## Scoring

### Response Parsing

Both services extract scores from the model response using:

```python
score = response[0].get('score', "no score key")
justification = response[0].get('justification', 'No justification provided')
```

| Condition | Result |
|---|---|
| `score` key present | Use the returned score and justification |
| `score` key missing | `score=0`, `justification="The evaluated [pair\|row] did not generate valid score and justification"` |
| Model call fails | `score=0`, `justification="Error during evaluation"` (or a more specific error message) |

### Statistics Computation

Statistics are computed from successful evaluations only (items where `score is not None`).

| Statistic | Computation |
|---|---|
| `average_score` | `sum(scores) / len(scores)`, rounded to 2 decimal places |
| `min_score` | `min(scores)` |
| `max_score` | `max(scores)` |
| `Overall_Average` | Recomputed at the top level from all collected scores |

When no successful evaluations exist, all statistics default to `0`.

---

## Output Format

### Legacy Evaluation Output

Saved as `qa_pairs_{model}_{timestamp}_evaluated.json`. Results are grouped by topic name.

```json
{
  "topic_name": {
    "average_score": 3.5,
    "min_score": 2,
    "max_score": 5,
    "evaluated_pairs": [
      {
        "question": "...",
        "solution": "...",
        "evaluation": {"score": 4, "justification": "..."}
      }
    ],
    "failed_pairs": [],
    "total_evaluated": 5,
    "total_failed": 0
  },
  "Overall_Average": 3.5
}
```

### Freeform Evaluation Output

Saved as `row_data_{model}_{timestamp}_evaluated.json`. Results are a flat structure.

```json
{
  "average_score": 7.5,
  "min_score": 5,
  "max_score": 10,
  "evaluated_rows": [
    {
      "row": {"field1": "value1", "...": "..."},
      "evaluation": {"score": 8, "justification": "..."}
    }
  ],
  "failed_rows": [],
  "total_evaluated": 10,
  "total_failed": 0,
  "Overall_Average": 7.5
}
```

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Individual evaluation failure | Error response (`score=0`) is returned; pipeline continues |
| `ModelHandlerError` raised | Re-raised immediately, stops all processing |
| Future result exception | Item added to `failed_pairs` / `failed_rows` with error message |
| Demo mode (`is_demo=true`) top-level failure | Raises `APIError` |
| Batch mode (`is_demo=false`) top-level failure | Updates job status to `ENGINE_FAILED` in the database, then re-raises |

Failed items are tracked separately from evaluated items. The `total_evaluated` and `total_failed` counters in the output reflect this split.

---

## Metadata Recording

After evaluation completes, both services persist metadata for the history and audit trail. The metadata is saved to the `evaluation_metadata` database table (via `DatabaseManager.save_evaluation_metadata()` in demo mode, or `DatabaseManager.update_job_evaluate()` in batch mode).

| Field | Description |
|---|---|
| `timestamp` | UTC ISO-8601 timestamp |
| `model_id` | LLM model identifier used for evaluation |
| `inference_type` | Provider (`aws_bedrock`, `CAII`, `openai`, etc.) |
| `caii_endpoint` | CAII endpoint URL (if applicable) |
| `use_case` | Use case enum value |
| `custom_prompt` | Evaluation rubric (custom or resolved default) |
| `model_parameters` | JSON-serialized `ModelParameters` |
| `generate_file_name` | Basename of the input data file |
| `evaluate_file_name` | Basename of the output evaluation file |
| `display_name` | Human-readable name for the UI |
| `local_export_path` | Full path to the saved evaluation JSON |
| `examples` | JSON-serialized few-shot examples |
| `Overall_Average` | Computed overall average score |
| `evaluation_type` | `"row"` (freeform only; legacy does not set this field) |

In batch mode, `update_job_evaluate()` writes the evaluate file name, output path, timestamp, overall average, and job status (`ENGINE_SUCCEEDED` or `ENGINE_FAILED`) to the job record, followed by `backup_and_restore_db()`.
