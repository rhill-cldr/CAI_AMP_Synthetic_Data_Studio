# History & CRUD Routes

Twelve endpoints manage generation/evaluation/export history, dataset preview, display-name edits, record deletion, and application upgrades. All list endpoints share a common pagination pattern; CRUD endpoints operate on individual records by `file_name`.

---

## Pagination

All history endpoints (`/generations/history`, `/evaluations/history`, `/exports/history`) share the same pagination pattern.

**Query parameters:**

| Parameter | Type | Default | Range | Description |
|---|---|---|---|---|
| page | int | 1 | >=1 | Page number |
| page\_size | int | 10 | 1--100 | Items per page |

**Pagination response:**

```json
{
  "data": [...],
  "pagination": {
    "total": 42,
    "page": 1,
    "page_size": 10,
    "total_pages": 5
  }
}
```

Total pages calculated as `(total + page_size - 1) // page_size`.

---

## GET /generations/history

List all generations with pagination. Before returning results, this endpoint checks for pending CML jobs by calling `get_job_status()` for each pending job ID and updating the database via `db_manager.update_job_statuses_generate()`. Uses `get_paginated_generate_metadata_light()` for the lightweight query.

**Query parameters:** standard pagination (`page`, `page_size`).

**Response:** paginated `data` array plus `pagination` object.

---

## GET /generations/\{file\_name\}

Get generation metadata for a specific file. Returns 404 if not found.

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_name | string | File identifier |

**Response:** generation metadata object.

**Errors:**

| Status | Condition |
|---|---|
| 404 | No generation found with the given filename |

---

## GET /dataset\_details/\{file\_path\}

Get dataset contents for preview. Behaviour depends on the file path:

| Path contains | Response key | Truncation |
|---|---|---|
| `qa_pairs` **and** `evaluated` | `{"evaluation": data}` | `evaluated_pairs` capped at 100 items per category |
| `qa_pairs` only | `{"generation": data[:100]}` | First 100 records |

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_path | string | Path to the dataset file |

---

## PUT /generations/display-name

Update the display name for a generation.

**Query parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_name | string | File identifier |
| display\_name | string | New display name |

**Response:** `{"message": "Display name updated successfully"}`

---

## DELETE /generations/\{file\_name\}

Delete generation metadata and optionally delete the local file.

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_name | string | File identifier |

**Query parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_path | string? | Optional path to delete local file |

**Response:** `{"message": "Generation data for '{file_name}' has been deleted successfully."}`

---

## GET /evaluations/history

List all evaluations with pagination. Same pending-job status update pattern as generations: checks pending evaluate job IDs via `get_pending_evaluate_job_ids()`, updates statuses with `update_job_statuses_evaluate()`, then returns paginated results from `get_paginated_evaluate_metadata()`.

**Query parameters:** standard pagination (`page`, `page_size`).

**Response:** paginated `data` array plus `pagination` object.

---

## GET /evaluations/\{file\_name\}

Get evaluation metadata for a specific file. Returns 404 if not found.

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_name | string | File identifier |

**Response:** evaluation metadata object.

**Errors:**

| Status | Condition |
|---|---|
| 404 | No evaluation found with the given filename |

---

## PUT /evaluations/display-name

Update the display name for an evaluation.

**Query parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_name | string | File identifier |
| display\_name | string | New display name |

**Response:** `{"message": "Display name updated successfully"}`

---

## DELETE /evaluations/\{file\_name\}

Delete evaluation metadata and optionally delete the local file.

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_name | string | File identifier |

**Query parameters:**

| Parameter | Type | Description |
|---|---|---|
| file\_path | string? | Optional path to delete local file |

**Response:** `{"message": "Evaluation data for '{file_name}' has been deleted successfully."}`

---

## GET /exports/history

List all exports with pagination. Same pending-job status update pattern: checks pending export job IDs via `get_pending_export_job_ids()`, updates statuses with `update_job_statuses_export()`, then returns paginated results from `get_paginated_export_metadata()`. Note: this endpoint is synchronous (`def`, not `async def`).

**Query parameters:** standard pagination (`page`, `page_size`).

**Response:** paginated `data` array plus `pagination` object.

---

## Upgrade Routes

### GET /synthesis-studio/check-upgrade

Check if application updates are available by comparing local and remote git commits. Runs `git fetch`, determines the current branch, then compares `git rev-parse` output for local vs. `origin/{branch}`.

**Response:** `StudioUpgradeStatus`

| Field | Type | Description |
|---|---|---|
| git\_local\_commit | string | Local HEAD commit hash |
| git\_remote\_commit | string | Remote HEAD commit hash |
| updates\_available | bool | `true` if commits differ |

### POST /synthesis-studio/upgrade

Perform a full application upgrade:

1. `git stash` then `git pull` then `git stash pop`
2. Run database migrations: `uv run python run_migrations.py`
3. Rebuild frontend: `bash build/shell_scripts/build_client.sh`
4. Restart CML application if any step succeeded (5-second delay)

**Response:** `StudioUpgradeResponse`

| Field | Type | Description |
|---|---|---|
| success | bool | Whether upgrade completed |
| message | string | Semicolon-separated status messages |
| git\_updated | bool | Whether git pull succeeded |
| frontend\_rebuilt | bool | Whether frontend rebuild succeeded |

---

## Route Summary

| Method | Path | Description |
|---|---|---|
| GET | `/generations/history` | List generations (paginated) |
| GET | `/generations/{file_name}` | Get generation by filename |
| GET | `/dataset_details/{file_path}` | Preview dataset contents |
| PUT | `/generations/display-name` | Rename a generation |
| DELETE | `/generations/{file_name}` | Delete a generation |
| GET | `/evaluations/history` | List evaluations (paginated) |
| GET | `/evaluations/{file_name}` | Get evaluation by filename |
| PUT | `/evaluations/display-name` | Rename an evaluation |
| DELETE | `/evaluations/{file_name}` | Delete an evaluation |
| GET | `/exports/history` | List exports (paginated) |
| GET | `/synthesis-studio/check-upgrade` | Check for updates |
| POST | `/synthesis-studio/upgrade` | Apply updates |
