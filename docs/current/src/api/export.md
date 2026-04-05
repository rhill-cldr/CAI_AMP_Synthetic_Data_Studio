# Export Routes

One endpoint handles exporting generated data to external destinations.

---

## POST /export\_results

Export generated data to HuggingFace and/or S3. In batch mode (`is_demo=false`), this creates a CML job via `synthesis_job.export_job()`.

### Request Body -- `Export_synth`

| Field | Type | Default | Description |
|---|---|---|---|
| export\_type | List\[str\] | `["huggingface"]` | Export destinations: `"huggingface"`, `"s3"` |
| file\_path | string | **required** | Path to the generated data file |
| display\_name | string? | `null` | Human-readable name (used as S3 key when `s3_config.key` is not set) |
| output\_key | string | `"Prompt"` | Column name for the prompt/input field |
| output\_value | string | `"Completion"` | Column name for the completion/output field |
| hf\_config | `HFConfig`? | `null` | HuggingFace export configuration |
| s3\_config | `S3Config`? | `null` | S3 export configuration |

### HFConfig

| Field | Type | Default | Description |
|---|---|---|---|
| hf\_repo\_name | string | **required** | Repository name on HuggingFace |
| hf\_username | string | **required** | HuggingFace username |
| hf\_token | string | **required** | HuggingFace API token |
| hf\_commit\_message | string | `"Hugging face export"` | Commit message |

### S3Config

| Field | Type | Default | Description |
|---|---|---|---|
| bucket | string | **required** | S3 bucket name |
| key | string | `""` | Object key (defaults to filename) |
| create\_if\_not\_exists | bool | `true` | Create bucket if it does not exist |

---

## Export Processing

The `Export_Service` handles the actual export after job dispatch.

### S3 export flow

1. Read `file_path`.
2. Determine key: use `s3_config.key`, or `display_name + ".json"`, or the original filename.
3. Call `export_to_s3()` with bucket, key, and create-bucket flag.
4. Update database with S3 path via `db.update_s3_path()`.

### HuggingFace export flow

1. Read JSON data from `file_path`.
2. Save HF token via `HfFolder.save_token()`.
3. Convert data to a HuggingFace `Dataset` using `output_key`/`output_value` columns. Features include a `Seeds` or `Generated_From` column depending on whether `doc_paths` were used.
4. Push to hub at `{hf_username}/{hf_repo_name}` with commit message.
5. Update database with HF URL via `db.update_hf_path()`.

---

## Example Request

```json
{
  "export_type": ["huggingface", "s3"],
  "file_path": "qa_pairs_claude_20241204_132411_test.json",
  "hf_config": {
    "hf_token": "your_token",
    "hf_username": "your_username",
    "hf_repo_name": "file_name",
    "hf_commit_message": "dataset trial"
  },
  "s3_config": {
    "bucket": "my-dataset-bucket",
    "create_if_not_exists": true
  }
}
```

---

## Error Handling

| Condition | Status | Error |
|---|---|---|
| S3 config missing for S3 export | 400 | `S3 configuration required for S3 export` |
| File not found | 404 | `File not found: {path}` |
| Invalid JSON | 400 | `Invalid JSON file: {detail}` |
| S3 export failure | 500 | `S3 export failed: {detail}` |
| Unexpected error | 500 | `Unexpected error: {detail}` |
