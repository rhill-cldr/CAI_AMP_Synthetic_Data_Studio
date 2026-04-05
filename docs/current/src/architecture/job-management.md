# Job Management (CML Jobs)

Synthetic Data Studio delegates long-running work to Cloudera Machine Learning (CML) Jobs. This chapter describes when jobs are created, how they are configured, and how the application tracks their lifecycle.

## When Jobs Are Used

The `is_demo` flag on each request controls the execution path:

| `is_demo` | Execution mode | Environment requirement |
|---|---|---|
| `true` | **Preview** -- synchronous, inline response | None (works locally) |
| `false` | **Batch** -- asynchronous CML Job | `CDSW_PROJECT_ID` env var must be set |

At startup, if `CDSW_PROJECT_ID` is present the application initializes a `SynthesisJob` instance backed by the CML API. If the variable is absent, `project_id` is set to `"local"` and all requests execute in preview mode regardless of `is_demo`.

For generation requests with document uploads (`doc_paths`), `is_demo` may be overridden to `false` when the total file size exceeds 1 GB (see [Resource Allocation](#resource-allocation)).

## SynthesisJob Class

`SynthesisJob` wraps all CML job creation, execution, and status retrieval.

**Constructor:**

```python
SynthesisJob(
    project_id: str,            # CDSW_PROJECT_ID
    client_cml: cmlapi.CMLAPIv2,
    path_manager: PathManager,
    db_manager: DatabaseManager,
    runtime_identifier: str     # From template job "Synthetic_data_base_job"
)
```

The `runtime_identifier` is resolved at startup by looking up a pre-existing CML job named `Synthetic_data_base_job` and reading its runtime identifier. This ensures all spawned jobs use the same container image and Python environment as the template.

**Source:** `app/services/synthesis_job.py`

## Job Creation Flow

All three job types follow the same path through `_create_and_run_job()`:

1. Serialize request parameters to a temporary JSON file (`{prefix}_args_{random_hex}.json`).
2. Build a `cmlapi.CreateJobRequest` with:
   - `script` -- one of `app/run_job.py`, `app/run_eval_job.py`, or `app/run_export_job.py`
   - `environment` -- `{"file_name": "<path to params JSON>"}`
   - `cpu` and `memory` -- resource allocation (see below)
   - `runtime_identifier` -- from the template job
3. Call `client_cml.create_job()` followed by `client_cml.create_job_run()`.
4. Save metadata (including `job_id`, `job_name`, `job_status`) to the SQLite database.
5. Return `{"job_name": ..., "job_id": ...}` to the caller.

The `generation_type` and `request_id` fields are embedded in the serialized JSON, not passed as separate environment variables.

## Job Types

| Method | Script | Purpose |
|---|---|---|
| `generate_job()` | `app/run_job.py` | Synthesis (SFT, Custom_Workflow, or Freeform) |
| `evaluate_job()` | `app/run_eval_job.py` | Evaluation of generated datasets |
| `export_job()` | `app/run_export_job.py` | Export to HuggingFace or S3 |

Each method serializes a different request model, saves technique-specific metadata, and delegates to `_create_and_run_job()`.

## Resource Allocation

Default allocation for all job types:

| Resource | Default |
|---|---|
| CPU cores | 2 |
| Memory (GB) | 4 |

For generation requests with document uploads, resource sizing scales with total file size. The `get_total_size()` function sums file sizes from the CML project filesystem and rounds up to the nearest GB (`math.ceil`).

| Condition | CPU | Memory | Behavior |
|---|---|---|---|
| `data_size <= 1` GB | 2 | 4 | Use `is_demo` as provided by client |
| `1 < data_size <= 10` GB | `max(2, data_size // 2)` | `data_size + 2` | Force `is_demo = false` (batch mode) |
| `data_size > 10` GB | -- | -- | Reject with HTTP 413 |

## Job Status Lifecycle

CML job runs progress through the following states:

```d2
direction: right

scheduling: ENGINE_SCHEDULING {
  shape: rectangle
  style.fill: "#fff3e0"
}

starting: ENGINE_STARTING {
  shape: rectangle
  style.fill: "#fff3e0"
}

running: ENGINE_RUNNING {
  shape: rectangle
  style.fill: "#e3f2fd"
}

succeeded: ENGINE_SUCCEEDED {
  shape: rectangle
  style.fill: "#e8f5e9"
}

failed: ENGINE_FAILED {
  shape: rectangle
  style.fill: "#ffebee"
}

timedout: ENGINE_TIMEDOUT {
  shape: rectangle
  style.fill: "#ffebee"
}

stopped: ENGINE_STOPPED {
  shape: rectangle
  style.fill: "#ffebee"
}

scheduling -> starting
starting -> running
running -> succeeded
running -> failed
running -> timedout
running -> stopped
```

The application also recognizes `ENGINE_STOPPING` and `ENGINE_UNKNOWN` as transient states.

## Status Polling

History endpoints (`/generations/history`, `/evaluations/history`, `/exports/history`) synchronize job statuses from CML as a side-effect. On each call:

1. Query the database for all records where `job_status != 'ENGINE_SUCCEEDED'` and `job_id IS NOT NULL`.
2. For each pending job, call `client_cml.list_job_runs(project_id, job_id, sort="-created_at", page_size=1)` to get the latest run status.
3. Batch-update the database with the current statuses.
4. Return the updated, paginated records.

This means the database converges to the true CML state only when a client fetches history. There is no background polling or webhook.

## Job Execution Scripts

All three scripts share the same bootstrap sequence: detect the virtual environment path, add it to `sys.path`, enable `nest_asyncio` for nested event loops, and read the params JSON from the path in the `file_name` environment variable. The params file is deleted after reading.

### run_job.py

Routes based on the `generation_type` field extracted from the params JSON:

| `generation_type` | Service | Method |
|---|---|---|
| `"freeform"` | `SynthesisService` | `generate_freeform()` |
| Other / absent, with `input_path` | `SynthesisLegacyService` | `generate_result()` (Custom_Workflow) |
| Other / absent, without `input_path` | `SynthesisLegacyService` | `generate_examples()` (SFT) |

On success the service writes results and updates `job_status` to `ENGINE_SUCCEEDED` in the database. On unhandled exception the script exits with code 1, causing CML to mark the run as `ENGINE_FAILED`.

### run_eval_job.py

Routes based on the same `generation_type` field:

| `generation_type` | Service | Method |
|---|---|---|
| `"freeform"` | `EvaluatorService` | `evaluate_row_data()` |
| Other / absent | `EvaluatorLegacyService` | `evaluate_results()` |

Status update behavior is the same as `run_job.py`.

### run_export_job.py

Does not branch on `generation_type`. Deserializes params into an `Export_synth` request and calls `Export_Service.export()`. The export service handles HuggingFace, S3, and local filesystem destinations based on the `export_type` field.

**Source files:**
- `app/run_job.py`
- `app/run_eval_job.py`
- `app/run_export_job.py`
