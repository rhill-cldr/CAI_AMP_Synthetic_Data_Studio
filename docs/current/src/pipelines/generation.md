# Generation Pipeline

Two service classes implement synthetic data generation. `SynthesisService` handles the Freeform technique. `SynthesisLegacyService` handles SFT and Custom\_Workflow. Both share the same high-level pattern: create a model handler, resolve topics, process topics concurrently, save results to a JSON file, and record metadata in SQLite.

| Service | Techniques | Entry Points | Source |
|---|---|---|---|
| `SynthesisService` | Freeform | `generate_freeform()` | `app/services/synthesis_service.py` |
| `SynthesisLegacyService` | SFT, Custom\_Workflow | `generate_examples()`, `generate_result()` | `app/services/synthesis_legacy_service.py` |

## Pipeline Flow

```d2
direction: right

request: Request {
  label: "SynthesisRequest"
}

handler: Create Handler {
  label: "create_handler()\nmodel_id + inference_type"
}

topics: Resolve Topics {
  label: "Topics from:\n- request.topics\n- doc_paths (chunked)\n- input_path (Custom_Workflow)"
}

pool: ThreadPoolExecutor {
  label: "ThreadPoolExecutor\nmax_concurrent_topics (default 5)"
}

process: Process Topic {
  label: "process_single_freeform()\nor process_single_topic()"
}

batch: Batch Prompt {
  label: "Build prompt\nQUESTIONS_PER_BATCH = 5"
}

llm: LLM Call {
  label: "model_handler\n.generate_response()"
}

validate: Validate {
  label: "Validate items\nFilter invalid"
}

fallback: Fallback {
  label: "Single-item\nprocessing"
}

save: Save Results {
  label: "JSON file +\nSQLite metadata"
}

request -> handler -> topics -> pool
pool -> process
process -> batch -> llm -> validate
validate -> fallback: invalid items {
  style.stroke-dash: 3
}
fallback -> llm
validate -> save: all valid
```

Steps common to all three techniques:

1. Create a `UnifiedModelHandler` via `create_handler()` with the request's `model_id`, `inference_type`, and optional `caii_endpoint`.
2. Resolve topics -- from `request.topics`, document chunks (`doc_paths`), or input files (`input_path`).
3. Submit topics to a `ThreadPoolExecutor` via `loop.run_in_executor()`.
4. Each topic is processed in batches of `QUESTIONS_PER_BATCH = 5`. Invalid items fall back to single-item processing.
5. Save the collected results to a JSON file.
6. Record metadata in the `generation_metadata` SQLite table.

---

## SFT Technique

**Service:** `SynthesisLegacyService`
**Entry point:** `generate_examples(request, job_name, is_demo)`

1. Creates model handler via `create_handler()`.
2. Resolves topics from `request.topics` or document chunks via `DocumentProcessor`.
3. For each topic, calls `process_single_topic()` via `ThreadPoolExecutor`.
4. Each topic processes in batches of `QUESTIONS_PER_BATCH = 5`.
5. Builds the prompt via `PromptBuilder.build_prompt()`.
6. Validates QA pairs: each must have `question` and `solution` string fields, both non-empty.
7. Falls back to single-item processing if any items in the batch are invalid.
8. Omit list tracks generated questions to avoid duplicates (kept to the last 100 entries).
9. Output format:

```json
[
  {"Seeds": "<topic>", "<output_key>": "<question>", "<output_value>": "<solution>"},
  ...
]
```

10. Saves to `qa_pairs_{model}_{timestamp}_{test|final}.json`.

### Validation Rule

```python
def _validate_qa_pair(self, pair: Dict) -> bool:
    return (
        isinstance(pair, dict) and
        "question" in pair and
        "solution" in pair and
        isinstance(pair["question"], str) and
        isinstance(pair["solution"], str) and
        len(pair["question"].strip()) > 0 and
        len(pair["solution"].strip()) > 0
    )
```

Source: `app/services/synthesis_legacy_service.py` -- `_validate_qa_pair()`.

---

## Custom\_Workflow Technique

**Service:** `SynthesisLegacyService`
**Entry point:** `generate_result(request, job_name, is_demo)`

1. Creates model handler with `custom_p=True` (returns raw text, not parsed JSON).
2. Reads inputs from `request.input_path` files, extracting the field named by `request.input_key`.
3. For each input, calls `process_single_input()` via `ThreadPoolExecutor` (hardcoded 5 workers).
4. Builds the prompt via `PromptBuilder.build_generate_result_prompt()`.
5. Response is raw text (not a JSON array).
6. Output format:

```json
[
  {"<input_key>": "<question>", "<output_value>": "<solution>"},
  ...
]
```

7. Saves to `qa_pairs_{model}_{timestamp}_{test|final}.json`.

### Differences from SFT

| Aspect | SFT | Custom\_Workflow |
|---|---|---|
| `custom_p` flag | `False` (JSON parsing) | `True` (raw text) |
| Input source | Topics list or document chunks | JSON files via `input_path` |
| Prompt builder | `build_prompt()` | `build_generate_result_prompt()` |
| Batch/fallback | Yes (batches of 5) | No (one LLM call per input) |
| Concurrency | Configurable via `max_concurrent_topics` | Hardcoded 5 workers |

Source: `app/services/synthesis_legacy_service.py` -- `generate_result()`, `process_single_input()`.

---

## Freeform Technique

**Service:** `SynthesisService`
**Entry point:** `generate_freeform(request, job_name, is_demo)`

1. Creates model handler via `create_handler()`.
2. Resolves topics the same way as SFT (from `request.topics` or document chunks).
3. For each topic, calls `process_single_freeform()` via `ThreadPoolExecutor`.
4. Uses the same batch/fallback strategy as SFT (`QUESTIONS_PER_BATCH = 5`).
5. Builds the prompt via `PromptBuilder.build_freeform_prompt()`.
6. Validates items: each must be a non-empty dict (no specific field requirements).
7. Omit list uses the first string field found, checking keys in this order: `id`, `name`, `title`, `question`, `prompt`, `key`. If none match, falls back to the first string value in the dict.
8. Output format:

```json
[
  {"Seeds": "<topic>", ...fields...},
  ...
]
```

When documents are the source, `"Seeds"` is replaced with `"Generated_From"`.

9. Saves to `freeform_data_{model}_{timestamp}_{test|final}.json`.

### Validation Rule

```python
def _validate_freeform_item(self, item: Dict) -> bool:
    return isinstance(item, dict) and len(item) > 0
```

Source: `app/services/synthesis_service.py` -- `_validate_freeform_item()`.

---

## Document Processing Integration

When `request.doc_paths` is provided, documents are chunked and used as topics. Both SFT and Freeform follow the same logic.

```python
processor = DocumentProcessor(chunk_size=1000, overlap=100)
for path in request.doc_paths:
    chunks = processor.process_document(path)
    topics.extend(chunks)
```

Topic and question allocation:

| Condition | Topics Used | Questions Per Topic |
|---|---|---|
| `num_questions <= len(chunks)` | First `num_questions` chunks | 1 |
| `num_questions > len(chunks)` | All chunks | `ceil(num_questions / len(chunks))` |

When documents are the source, the output key changes from `"Seeds"` to `"Generated_From"` in the saved JSON.

Source: `app/services/doc_extraction.py` -- `DocumentProcessor`.

---

## Concurrency

| Parameter | Default | Range | Notes |
|---|---|---|---|
| `max_concurrent_topics` | 5 | 1--100 | Configurable per request (SFT, Freeform) |
| Custom\_Workflow workers | 5 | -- | Hardcoded in `generate_result()` |
| `QUESTIONS_PER_BATCH` | 5 | -- | Hardcoded class constant |

Both services use `ThreadPoolExecutor` with `loop.run_in_executor()` to bridge synchronous topic processing into the async FastAPI request lifecycle.

---

## Error Handling

Error behavior differs between the two service classes.

### Freeform (`SynthesisService`)

- Errors within `process_single_freeform()` are collected into a per-topic error list, never raised.
- Partial results are saved even when some topics fail.
- After saving, if any `ModelHandlerError` occurred, the first critical error is raised as `APIError`.
- In batch mode (`is_demo=false`), failures update `job_status` to `ENGINE_FAILED`.

### Legacy (`SynthesisLegacyService`)

- `ModelHandlerError` in `process_single_topic()` propagates up (re-raised).
- `JSONParsingError` (a subclass of `ModelHandlerError`) falls back to single-item processing instead of propagating.
- In single-item fallback, errors are collected rather than raised.
- `generate_result()` raises `APIError` on any `ModelHandlerError`.

### Common Behavior

| Mode | On Error |
|---|---|
| Preview (`is_demo=true`) | `APIError` raised, caught by middleware, returned as `{"status": "failed", "error": "..."}` |
| Batch (`is_demo=false`) | `job_status` updated to `ENGINE_FAILED` in SQLite; exception re-raised for CML to mark the job run as failed |

---

## Metadata Recording

Both services save to the `generation_metadata` table via `DatabaseManager`. The following fields are serialized as JSON strings before storage:

| Field | Serialization |
|---|---|
| `model_parameters` | `json.dumps(model_params.model_dump())` |
| `topics` | `json.dumps(topics)` (null when using `doc_paths`) |
| `examples` | `json.dumps(examples)` |
| `schema` | `json.dumps(schema)` |
| `doc_paths` | `json.dumps(doc_paths)` |
| `input_path` | `json.dumps(input_path)` |

In preview mode, metadata is saved via `save_generation_metadata()`. In batch mode, the job row is updated via `update_job_generate()` with the file name, output path, timestamp, and final job status. Batch mode also triggers `backup_and_restore_db()` to persist the SQLite database.

Source: `app/services/synthesis_service.py`, `app/services/synthesis_legacy_service.py`.
