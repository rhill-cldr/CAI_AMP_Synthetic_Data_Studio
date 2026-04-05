# Database Schema

Synthetic Data Studio persists all generation, evaluation, and export metadata in a single SQLite database. The `DatabaseManager` class in `app/core/database.py` owns the schema, connection lifecycle, and all CRUD operations.

---

## Overview

| Setting | Value | Purpose |
|---|---|---|
| Database file | `metadata.db` in project root | Single-file persistence |
| `journal_mode` | `WAL` | Write-Ahead Logging for concurrent read/write |
| `busy_timeout` | `60000` (60 s) | Prevents immediate `SQLITE_BUSY` failures |
| `synchronous` | `FULL` | Guarantees durability after commit |
| `foreign_keys` | `ON` | Enables foreign key enforcement |
| Connection timeout | `60` seconds | Python-level `sqlite3.connect()` timeout |

On startup, `init_db()` runs `PRAGMA quick_check` against an existing database. If the check raises `sqlite3.DatabaseError`, the database file and its WAL/SHM companions are deleted and recreated from scratch. If table creation itself fails, the database file is removed and `init_db()` retries once before propagating the exception.

---

## Entity-Relationship Diagram

```d2
generation_metadata: generation_metadata {
  shape: sql_table
  id: INTEGER {constraint: PK AUTOINCREMENT}
  timestamp: TEXT
  technique: TEXT
  model_id: TEXT
  inference_type: TEXT
  caii_endpoint: TEXT
  use_case: TEXT
  custom_prompt: TEXT
  model_parameters: TEXT
  input_key: TEXT
  output_key: TEXT
  output_value: TEXT
  generate_file_name: TEXT {constraint: UNIQUE}
  display_name: TEXT
  local_export_path: TEXT
  hf_export_path: TEXT
  s3_export_path: TEXT
  num_questions: FLOAT
  total_count: FLOAT
  topics: TEXT
  examples: TEXT
  schema: TEXT
  doc_paths: TEXT
  input_path: TEXT
  job_id: TEXT
  job_name: TEXT {constraint: UNIQUE}
  job_status: TEXT
  job_creator_name: TEXT
  completed_rows: INTEGER
}

evaluation_metadata: evaluation_metadata {
  shape: sql_table
  id: INTEGER {constraint: PK AUTOINCREMENT}
  timestamp: TEXT
  model_id: TEXT
  inference_type: TEXT
  caii_endpoint: TEXT
  use_case: TEXT
  custom_prompt: TEXT
  model_parameters: TEXT
  generate_file_name: TEXT
  evaluate_file_name: TEXT {constraint: UNIQUE}
  display_name: TEXT
  local_export_path: TEXT
  examples: TEXT
  average_score: FLOAT
  job_id: TEXT
  job_name: TEXT {constraint: UNIQUE}
  job_status: TEXT
  job_creator_name: TEXT
}

export_metadata: export_metadata {
  shape: sql_table
  id: INTEGER {constraint: PK AUTOINCREMENT}
  timestamp: TEXT
  display_export_name: TEXT
  display_name: TEXT
  local_export_path: TEXT
  hf_export_path: TEXT
  s3_export_path: TEXT
  job_id: TEXT
  job_name: TEXT {constraint: UNIQUE}
  job_status: TEXT
  job_creator_name: TEXT
}

evaluation_metadata.generate_file_name -> generation_metadata.generate_file_name: references
```

The `evaluation_metadata.generate_file_name` column references `generation_metadata.generate_file_name`, linking each evaluation back to the dataset it scored. This relationship is maintained by application logic rather than a SQL `FOREIGN KEY` constraint.

---

## Table Details

### generation\_metadata

Stores one row per generation job. The primary record for all synthetic data produced by the system.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK AUTOINCREMENT | Auto-incrementing row identifier |
| timestamp | TEXT | | ISO-format creation or completion time |
| technique | TEXT | | `SFT`, `Custom_Workflow`, or `Freeform` |
| model\_id | TEXT | | LLM model identifier used for generation |
| inference\_type | TEXT | | Provider type (e.g., `caii`, `bedrock`, `openai`) |
| caii\_endpoint | TEXT | | Cloudera AI Inference endpoint URL |
| use\_case | TEXT | | Use case category |
| custom\_prompt | TEXT | | Final prompt sent to the model |
| model\_parameters | TEXT | | **JSON-serialized.** Temperature, top\_p, and other model params |
| input\_key | TEXT | | Schema key for input column |
| output\_key | TEXT | | Schema key for output column |
| output\_value | TEXT | | Schema key for output value column |
| generate\_file\_name | TEXT | UNIQUE | Output filename, serves as the logical primary key for lookups |
| display\_name | TEXT | | User-facing name shown in the UI |
| local\_export\_path | TEXT | | Filesystem path to the generated dataset |
| hf\_export\_path | TEXT | | HuggingFace Hub export path (set after export) |
| s3\_export\_path | TEXT | | S3 export path (set after export) |
| num\_questions | FLOAT | | Requested number of items per topic |
| total\_count | FLOAT | | Total items requested |
| topics | TEXT | | **JSON-serialized.** List of generation topics |
| examples | TEXT | | **JSON-serialized.** Few-shot examples provided by the user |
| schema | TEXT | | **JSON-serialized.** Output schema definition |
| doc\_paths | TEXT | | **JSON-serialized.** Uploaded document paths for RAG-style generation |
| input\_path | TEXT | | **JSON-serialized.** Seed input file path (Custom\_Workflow) |
| job\_id | TEXT | | CML Job ID (null for preview mode) |
| job\_name | TEXT | UNIQUE | CML Job name (null for preview mode) |
| job\_status | TEXT | | CML Job run status (e.g., `ENGINE_SUCCEEDED`) |
| job\_creator\_name | TEXT | | Username that created the job |
| completed\_rows | INTEGER | | Number of rows successfully generated |

**JSON-serialized fields:** `model_parameters`, `topics`, `examples`, `schema`, `doc_paths`, `input_path`. These columns store TEXT but contain JSON strings. Read operations deserialize them with `json.loads()` before returning results to callers.

### evaluation\_metadata

Stores one row per evaluation job. Each evaluation scores an existing generation dataset.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK AUTOINCREMENT | Auto-incrementing row identifier |
| timestamp | TEXT | | ISO-format creation or completion time |
| model\_id | TEXT | | LLM model identifier used for evaluation |
| inference\_type | TEXT | | Provider type |
| caii\_endpoint | TEXT | | Cloudera AI Inference endpoint URL |
| use\_case | TEXT | | Use case category |
| custom\_prompt | TEXT | | Evaluation prompt |
| model\_parameters | TEXT | | **JSON-serialized.** Model parameters |
| generate\_file\_name | TEXT | | References `generation_metadata.generate_file_name` |
| evaluate\_file\_name | TEXT | UNIQUE | Output filename for evaluation results |
| display\_name | TEXT | | User-facing name shown in the UI |
| local\_export\_path | TEXT | | Filesystem path to the evaluation results |
| examples | TEXT | | **JSON-serialized.** Few-shot examples for evaluation |
| average\_score | FLOAT | | Mean evaluation score across all items |
| job\_id | TEXT | | CML Job ID |
| job\_name | TEXT | UNIQUE | CML Job name |
| job\_status | TEXT | | CML Job run status |
| job\_creator\_name | TEXT | | Username that created the job |

**JSON-serialized fields:** `model_parameters`, `examples`.

### export\_metadata

Stores one row per export job. Tracks exports to HuggingFace Hub, S3, or the local filesystem.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK AUTOINCREMENT | Auto-incrementing row identifier |
| timestamp | TEXT | | ISO-format creation time |
| display\_export\_name | TEXT | | User-facing export name |
| display\_name | TEXT | | Display name of the source dataset |
| local\_export\_path | TEXT | | Local filesystem export path |
| hf\_export\_path | TEXT | | HuggingFace Hub export path |
| s3\_export\_path | TEXT | | S3 export path |
| job\_id | TEXT | | CML Job ID |
| job\_name | TEXT | UNIQUE | CML Job name |
| job\_status | TEXT | | CML Job run status |
| job\_creator\_name | TEXT | | Username that created the job |

This table contains no JSON-serialized fields.

### UNIQUE Constraints

Each table enforces uniqueness on file name and job name columns to prevent duplicate records:

| Table | UNIQUE Column(s) |
|---|---|
| generation\_metadata | `generate_file_name`, `job_name` |
| evaluation\_metadata | `evaluate_file_name`, `job_name` |
| export\_metadata | `job_name` |

---

## CRUD Operations

### Writes

| Method | Table | Operation | Notes |
|---|---|---|---|
| `save_generation_metadata()` | generation\_metadata | INSERT | `BEGIN IMMEDIATE` transaction; returns `lastrowid` |
| `save_evaluation_metadata()` | evaluation\_metadata | INSERT | `BEGIN IMMEDIATE` transaction; computes `average_score` from `Overall_Average` |
| `save_export_metadata()` | export\_metadata | INSERT | `BEGIN IMMEDIATE` transaction; returns `lastrowid` |

### Updates

| Method | Table | Operation | Notes |
|---|---|---|---|
| `update_job_generate()` | generation\_metadata | UPDATE by `job_name` | 3 retries with 1 s delay; sets `generate_file_name`, `local_export_path`, `timestamp`, `job_status`, `completed_rows` |
| `update_job_evaluate()` | evaluation\_metadata | UPDATE by `job_name` | 3 retries with 1 s delay; sets `evaluate_file_name`, `local_export_path`, `timestamp`, `average_score`, `job_status` |
| `update_hf_path()` | generation\_metadata | UPDATE `hf_export_path` | By `generate_file_name` |
| `update_s3_path()` | generation\_metadata | UPDATE `s3_export_path` | By `generate_file_name` |

### Reads

| Method | Table | Operation | Notes |
|---|---|---|---|
| `get_paginated_generate_metadata()` | generation\_metadata | SELECT with LIMIT/OFFSET | Deserializes 6 JSON fields; returns `(total_count, results)` |
| `get_paginated_generate_metadata_light()` | generation\_metadata | SELECT subset with LIMIT/OFFSET | Lightweight query for list views; no JSON deserialization |
| `get_paginated_evaluate_metadata()` | evaluation\_metadata | SELECT with LIMIT/OFFSET | Deserializes 2 JSON fields; returns `(total_count, results)` |
| `get_paginated_export_metadata()` | export\_metadata | SELECT with LIMIT/OFFSET | No JSON fields; returns `(total_count, results)` |
| `get_metadata_by_filename()` | generation\_metadata | SELECT by `generate_file_name` | Deserializes JSON fields; returns single dict or `None` |
| `get_evaldata_by_filename()` | evaluation\_metadata | SELECT by `evaluate_file_name` | Deserializes JSON fields; returns single dict or `None` |
| `get_all_generate_metadata()` | generation\_metadata | SELECT all | Full table scan, ordered by timestamp DESC |
| `get_all_evaluate_metadata()` | evaluation\_metadata | SELECT all | Full table scan, ordered by timestamp DESC |
| `get_all_export_metadata()` | export\_metadata | SELECT all | Full table scan, ordered by timestamp DESC |

### Deletes

| Method | Table | Operation | Notes |
|---|---|---|---|
| `delete_generate_data()` | generation\_metadata | DELETE by `generate_file_name` | Also removes the local file at `file_path` if provided and exists |
| `delete_evaluate_data()` | evaluation\_metadata | DELETE by `evaluate_file_name` | Also removes the local file at `file_path` if provided and exists |

### Maintenance

| Method | Table | Operation | Notes |
|---|---|---|---|
| `backup_and_restore_db()` | all | WAL checkpoint, backup, delete, restore | Checkpoints WAL, creates backup via `sqlite3.backup()`, removes original, restores from backup, deletes backup file |

---

## Concurrency

All write operations acquire an exclusive lock immediately via `BEGIN IMMEDIATE` transactions. This prevents write starvation by failing fast when another writer holds the lock, rather than waiting and potentially deadlocking.

**Retry mechanism.** The `update_job_generate()` and `update_job_evaluate()` methods retry up to 3 times with a 1-second delay between attempts. Retries handle two cases:

1. **Database locked** -- another transaction holds the write lock (`sqlite3.OperationalError` containing `"database is locked"`).
2. **Row not yet available** -- the INSERT from job creation has not yet committed (row count is 0 for the given `job_name`).

**WAL mode** enables readers to proceed concurrently with a single writer. Multiple connections can read the database while a write transaction is in progress; reads see the last committed state.

**Busy timeout** (`60000` ms) provides an additional safety net at the SQLite engine level, causing `sqlite3.connect()` to wait up to 60 seconds for a lock rather than failing immediately.

---

## Job Status Tracking

Three pairs of methods synchronize job status from CML into the database:

| Pending query | Bulk update | Table |
|---|---|---|
| `get_pending_generate_job_ids()` | `update_job_statuses_generate()` | generation\_metadata |
| `get_pending_evaluate_job_ids()` | `update_job_statuses_evaluate()` | evaluation\_metadata |
| `get_pending_export_job_ids()` | `update_job_statuses_export()` | export\_metadata |

**Pending detection.** Each `get_pending_*_job_ids()` method selects all `job_id` values where `job_status != 'ENGINE_SUCCEEDED'` and `job_id IS NOT NULL`.

**Batch update.** Each `update_job_statuses_*()` method accepts a `Dict[str, str]` mapping `job_id` to its new status. It constructs a single `UPDATE ... SET job_status = CASE WHEN job_id = ? THEN ? ... END WHERE job_id IN (...)` statement, updating all rows in one transaction. This avoids N separate UPDATE statements when many jobs are pending.

These methods are called by the history endpoints (`/generations/history`, `/evaluations/history`, `/exports/history`) as a side-effect before returning paginated results. See [History & CRUD Routes](../api/history.md) for details.

---

**Source:** `app/core/database.py`
