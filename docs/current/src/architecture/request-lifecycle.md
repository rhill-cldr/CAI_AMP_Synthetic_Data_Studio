# Request Lifecycle

Every HTTP request passes through the global middleware before reaching its route handler. The middleware enforces per-endpoint timeouts and provides a uniform error response envelope.

## Middleware Pipeline

Request processing order:

1. **CORS middleware** -- `allow_origins=["*"]`, all methods, all headers, credentials allowed.
2. **Global timeout middleware** -- wraps `call_next` in `asyncio.wait_for()` with a per-endpoint timeout from `get_timeout_for_request()`.
3. **Error wrapping** -- catches `APIError`, `InvalidModelError`, `RuntimeError`, and generic `Exception`; returns `{"status": "failed", "error": "..."}` with the appropriate HTTP status code.

Source: `app/main.py` -- `global_middleware()`, `get_timeout_for_request()`.

## Timeout Configuration

Timeouts are resolved by matching the request path in `get_timeout_for_request()`.

| Endpoint Pattern | Timeout (seconds) | Rationale |
|---|---|---|
| `*/generate` | 200 | LLM generation round-trip |
| `*/freeform` | 300 | Freeform generation with larger payloads |
| `*/create_custom_prompt` | 300 | Prompt generation via LLM |
| `*/evaluate` | 200 | Evaluation round-trip |
| `*/export_results` | 200 | Export serialisation |
| `*health*` | 5 | Liveness probe |
| `*/upgrade` | 2000 | Git pull + frontend rebuild + app restart |
| All others | 60 | Default |

## Preview Mode (`is_demo=true`)

Results are returned synchronously in the response body. Used for small runs (typically 25 items or fewer) or local development where `CDSW_PROJECT_ID` is unset.

```d2
direction: right

client: Client
middleware: FastAPI Middleware
handler: Route Handler
service: "Service\n(generate_*/evaluate_*)"
model: "UnifiedModelHandler\n.generate_response"
llm: LLM Provider
parse: JSON Parse
save_file: Save File
save_meta: Save Metadata to DB
response: Return JSON Response

client -> middleware -> handler -> service -> model -> llm
llm -> parse -> save_file -> save_meta -> response
response -> client
```

Key characteristics:

- The entire pipeline executes within the middleware timeout.
- The response body contains the generated or evaluated data directly.
- Database metadata is written before the response returns.

## Batch Mode (`is_demo=false`)

Available only when running inside a Cloudera ML environment (`CDSW_PROJECT_ID` is set). Used for large datasets. The client polls history endpoints for completion.

### Job Submission

```d2
direction: right

client: Client
handler: Route Handler
job: "SynthesisJob\n.generate_job"
cml_api: "CML API\n(create_job + run)"
save_meta: "Save Metadata\n(job_status=pending)"
resp: "Return\n{job_name, job_id}"

client -> handler -> job -> cml_api -> save_meta -> resp
resp -> client
```

### Async Worker Execution

```d2
direction: right

worker: CML Worker
run_job: run_job.py
service: "Service\n.generate_*"
update: "Update Metadata\n(job_status=succeeded|failed)"

worker -> run_job -> service -> update
```

### Client Polling

```d2
direction: right

client: Client
history: "GET /generations/history"
db: Database
status: "Check job_status field"

client -> history -> db -> status -> client
```

Job lifecycle:

1. `SynthesisJob.generate_job()` serialises the request to a JSON args file.
2. `cmlapi.CreateJobRequest` creates a CML job pointing at `app/run_job.py` (or `app/run_eval_job.py`).
3. `create_job_run()` starts execution. Metadata is saved with `job_status=pending`.
4. The route returns `{job_name, job_id}` immediately.
5. The CML worker reads the args file, instantiates the appropriate service, and runs to completion.
6. On completion the worker updates the database row to `succeeded` or `failed`.
7. `GET /generations/history` (or `/evaluations/history`, `/exports/history`) checks for pending job IDs, polls CML for current status, updates the DB, and returns paginated results.

## Error Handling

### Exception Hierarchy

```
Exception
  └── APIError (status_code=400)
        ├── ModelHandlerError
        │     └── InvalidModelError (status_code=404)
        └── JSONParsingError (status_code=422)
```

Source: `app/core/exceptions.py`.

### HTTP Status Code Mapping

| Code | Meaning | Trigger |
|---|---|---|
| 400 | Bad Request | Invalid input, missing fields, `RuntimeError` |
| 404 | Not Found | Invalid model ID, resource not found |
| 413 | Payload Too Large | Dataset exceeds 10 GB |
| 422 | Unprocessable Entity | JSON parsing failure from LLM response |
| 500 | Internal Server Error | Unexpected exceptions |
| 504 | Gateway Timeout | Operation exceeds endpoint timeout |

### `InvalidModelError` Subtypes

| `error_subtype` | Condition |
|---|---|
| `throughput_configuration_error` | Model requires an inference profile (on-demand throughput not supported) |
| `invalid_identifier_error` | Model ID not recognised by the provider |
| `general_model_error` | Other model configuration issues |

All error responses share the envelope:

```json
{"status": "failed", "error": "<message>"}
```
