# Export Destinations

Synthetic Data Studio supports three export destinations: the local filesystem, HuggingFace Hub, and Amazon S3. Every generation or evaluation result is written to disk first; HuggingFace and S3 exports occur as secondary steps after the local file exists.

---

## Local Filesystem

All generation and evaluation results are automatically saved to the local filesystem. This is not an optional export -- it is the first step in every pipeline. S3 and HuggingFace exports read from the local file after it is written.

| Aspect | Detail |
|---|---|
| Base directory | Project root, or `synthetic-data-studio/` prefix when `IS_COMPOSABLE=true` |
| Path management | `PathManager` resolves the prefix based on the `IS_COMPOSABLE` environment variable |
| Filename patterns | See [Output File Formats](./file-formats.md) |
| Metadata tracking | Local path is recorded in `generation_metadata.local_export_path` or `evaluation_metadata.local_export_path` |

No additional configuration is required for local export.

---

## HuggingFace Hub

The `Export_Service.export()` method handles HuggingFace export when `"huggingface"` is present in the `export_type` list.

### Flow

1. Read JSON data from `file_path`.
2. Save the HuggingFace token via `HfFolder.save_token(hf_config.hf_token)`.
3. Look up the generation metadata via `db.get_metadata_by_filename()` to check whether `doc_paths` was set.
4. Build a HuggingFace `Dataset` with explicit `Features`:
   - If `doc_paths` was set: columns are `Generated_From`, `{output_key}`, `{output_value}`.
   - If `doc_paths` was not set: columns are `Seeds`, `{output_key}`, `{output_value}`.
   - All columns use `Value('string')` type.
5. Push to hub at `{hf_username}/{hf_repo_name}` with the configured commit message.
6. Update the database with the HuggingFace URL via `db.update_hf_path()`.

### Configuration -- HFConfig

| Field | Type | Default | Description |
|---|---|---|---|
| hf\_repo\_name | string | **required** | Repository name on HuggingFace |
| hf\_username | string | **required** | HuggingFace username |
| hf\_token | string | **required** | HuggingFace API token |
| hf\_commit\_message | string | `"Hugging face export"` | Commit message for the push |

### Dependencies

| Package | Imports used |
|---|---|
| `huggingface_hub` | `HfApi`, `HfFolder` |
| `datasets` | `Dataset`, `Features`, `Value` |

---

## Amazon S3

The `export_to_s3()` function in `app/services/s3_export.py` handles S3 uploads.

### Flow

1. Verify `file_path` exists on disk.
2. Resolve AWS credentials: use provided `access_key`/`secret_key` parameters, or fall back to `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_DEFAULT_REGION` environment variables.
3. Create a boto3 S3 client with the resolved credentials and optional region.
4. Determine the S3 object key:
   - Use `s3_config.key` if provided.
   - Else use `display_name + ".json"` if `display_name` is set.
   - Else use the original filename via `os.path.basename(file_path)`.
5. Check if the bucket exists via `head_bucket()`:
   - If 404 and `create_bucket=True`: create the bucket (with `LocationConstraint` for non-`us-east-1` regions).
   - If 404 and `create_bucket=False`: raise an error.
   - Other errors: raise an error.
6. Upload the file via `s3_client.upload_file()` with `ContentType: application/json`.
7. Return `{"s3": "s3://{bucket}/{key}"}`.
8. The calling `Export_Service` updates the database with the S3 path via `db.update_s3_path()`.

### Configuration -- S3Config

| Field | Type | Default | Description |
|---|---|---|---|
| bucket | string | **required** | S3 bucket name |
| key | string | `""` | Object key; defaults to filename if empty |
| create\_if\_not\_exists | bool | `true` | Create the bucket if it does not exist |

### AWS Credential Resolution

| Priority | Source |
|---|---|
| 1 | `access_key`/`secret_key` parameters passed to `export_to_s3()` |
| 2 | `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` environment variables |

Region is resolved from the `region` parameter or the `AWS_DEFAULT_REGION` environment variable.

---

## Error Handling

| Destination | Error | HTTP Status | Detail |
|---|---|---|---|
| S3 | Config missing | 400 | `S3 configuration required for S3 export` |
| S3 | File not found | ValueError | `File not found: {path}` |
| S3 | Credentials missing | ValueError | `AWS credentials not provided and not found in environment variables` |
| S3 | Bucket doesn't exist (create=false) | ValueError | `Bucket {name} does not exist and create_bucket is False` |
| S3 | Upload failure | ValueError | `Error uploading file to S3: {detail}` |
| HuggingFace | File not found | 404 | `File not found: {path}` |
| HuggingFace | Invalid JSON | 400 | `Invalid JSON file: {detail}` |
| Any | Unexpected | 500 | Wrapped in `APIError` |

---

## Database Updates After Export

| Export Type | Database Method | Column Updated |
|---|---|---|
| S3 | `db.update_s3_path(filename, s3_path)` | `generation_metadata.s3_export_path` |
| HuggingFace | `db.update_hf_path(filename, hf_url)` | `generation_metadata.hf_export_path` |

Both methods are called by `Export_Service.export()` immediately after a successful upload. The `filename` argument is `os.path.basename(request.file_path)`.

---

**Source:** `app/services/export_results.py`, `app/services/s3_export.py`, `app/models/request_models.py`
