# Model Alignment (DPO/KTO)

Source: `app/services/model_alignment.py`

The `ModelAlignment` class produces DPO (Direct Preference Optimization) and KTO (Kahneman-Tversky Optimization) datasets from existing prompt-completion pairs. It works by generating alternate completions, scoring both the original and alternate versions, and constructing preference pairs based on which completion scored higher.

`ModelAlignment` composes two legacy services internally:

| Dependency | Role |
|---|---|
| `SynthesisLegacyService` | Generates alternate completions via `process_single_input()` |
| `EvaluatorLegacyService` | Scores completions via `evaluate_single_pair()` |

Both services use the legacy prompt construction paths (`build_generate_result_prompt`, `build_eval_prompt`).

---

## Pipeline Diagram

```d2
direction: right

input: Input Data {
  label: "Input JSON\n[{Prompt, Completion}, ...]"
}

alt: Generate Alternates {
  label: "alternate_result()\nThreadPoolExecutor(5)"
}

eval_orig: Score Original {
  label: "evaluate_generation()\noriginal completions"
}

eval_alt: Score Alternate {
  label: "evaluate_generation()\nalternate completions"
}

compare: Compare Scores {
  label: "Higher score = chosen\nLower score = rejected"
}

dpo: DPO Output {
  label: "DPO pairs\nchosen + rejected"
}

kto: KTO Output {
  label: "KTO entries\nLabel: true/false"
}

input -> alt -> eval_orig
alt -> eval_alt
eval_orig -> compare
eval_alt -> compare
compare -> dpo
compare -> kto
```

---

## Pipeline Steps

### Step 1: Generate Alternate Completions

Method: `alternate_result(synthesis_request)`

Reads the input file specified in `synthesis_request.input_path[0]` and extracts prompts (`output_key`) and original responses (`output_value`) from the JSON array. A model handler is created with `custom_p=True` so that raw text is returned rather than parsed JSON.

Each prompt is submitted to `SynthesisLegacyService.process_single_input()` via a `ThreadPoolExecutor` with 5 workers. The executor delegates to `asyncio.run()` inside each thread so the async service method can execute in the thread pool context.

The return value is a list of dictionaries:

| Field | Value |
|---|---|
| `{output_key}` | The original prompt |
| `{output_value}` | The original completion |
| `Alternate_Completion` | The newly generated completion |

### Step 2: Score Both Versions

Method: `evaluate_generation(result, output_key, output_value, evaluation_request)`

Called twice -- once for original completions and once for alternates. Creates an evaluation model handler (without `custom_p`), builds QA pairs from the result list using the specified `output_key` and `output_value`, then evaluates all pairs in parallel via `EvaluatorLegacyService.evaluate_single_pair()` with a `ThreadPoolExecutor` of 4 workers.

| Invocation | `output_key` | `output_value` | Scores |
|---|---|---|---|
| First call | original prompt key | original completion key | `scores_initial` |
| Second call | original prompt key | `"Alternate_Completion"` | `scores_alternate` |

Each call returns a list of numeric scores extracted from the evaluation results.

### Step 3: Build Preference Pairs

Method: `model_alignment(synthesis_request, evaluation_request, job_name, is_demo)`

This is the main entry point. It orchestrates all three steps and produces two output formats from the scored results.

**DPO construction:** For each item at index `idx`, compare `scores_initial[idx]` against `scores_alternate[idx]`. The higher-scored completion becomes `chosen_response`; the lower-scored completion becomes `rejected_response`.

**KTO construction:** Each DPO pair is flattened into two KTO entries -- one for the chosen response (`Label: true`) and one for the rejected response (`Label: false`).

---

## Output Formats

### DPO Schema

| Field | Type | Description |
|---|---|---|
| `instruction` | string | The input prompt |
| `chosen_response` | string | Higher-scored completion |
| `rejected_response` | string | Lower-scored completion |
| `chosen_rating` | float | Score assigned to the chosen response |
| `rejected_rating` | float | Score assigned to the rejected response |

```json
{
  "instruction": "Write a Python function...",
  "chosen_response": "def foo(): ...",
  "rejected_response": "Here is a function...",
  "chosen_rating": 4,
  "rejected_rating": 2
}
```

### KTO Schema

| Field | Type | Description |
|---|---|---|
| `Prompt` | string | The input prompt |
| `Completion` | string | The response text |
| `Label` | bool | `true` for chosen, `false` for rejected |
| `Rating` | float | Evaluation score |

```json
[
  {"Prompt": "Write...", "Completion": "def foo()...", "Label": true, "Rating": 4},
  {"Prompt": "Write...", "Completion": "Here is...", "Label": false, "Rating": 2}
]
```

### Return Value

```json
{
  "dpo": [/* DPO pairs */],
  "kto": [/* KTO entries */]
}
```

KTO always contains exactly 2x the entries of DPO (one chosen entry + one rejected entry per pair).

---

## Concurrency Configuration

| Step | Executor | `max_workers` | Method |
|---|---|---|---|
| Generate alternates | `ThreadPoolExecutor` | 5 | `asyncio.run()` inside each thread via `loop.run_in_executor()` |
| Score completions | `ThreadPoolExecutor` | 4 | `executor.submit()` with `as_completed()` |

The generation step uses `asyncio.gather()` to await all thread futures concurrently. The evaluation step collects results as they complete via `concurrent.futures.as_completed()`.

---

## Error Handling

| Condition | Behavior |
|---|---|
| Any exception in `alternate_result()` | Wrapped in `APIError` with message `"Model alignment failed: {detail}"` |
| Individual evaluation failure | Logged; appended to `failed_pairs` list; does not halt the batch |
| Score extraction | Only pairs with a non-`None` score are included in the returned score list |

All errors are logged to `logs/model_alignment.log` (general) and `logs/model_alignment_errors.log` (errors only), both configured as `RotatingFileHandler` with 10 MB rotation and 5 backups.

There is no batch mode support for alignment -- it always runs synchronously in preview mode.

---

## Initialization

`ModelAlignment.__init__()` creates its service dependencies and supporting infrastructure:

| Attribute | Source |
|---|---|
| `self.synthesis_service` | `SynthesisLegacyService()` |
| `self.evaluator_service` | `EvaluatorLegacyService()` |
| `self.db` | `DatabaseManager()` |
| `self.bedrock_client` | `get_bedrock_client()` |
| `self.logger` | `logging.getLogger('model_alignment')` via `_setup_logging()` |

Source: `app/services/model_alignment.py`
