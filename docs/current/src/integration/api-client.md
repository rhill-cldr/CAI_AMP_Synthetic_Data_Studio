# Building an API Client

This chapter walks through building a Python client that drives the generate, evaluate, and export workflow against the SDS REST API.

---

## Base URL and Authentication

The API is served on port **8100** (configurable via the `CDSW_APP_PORT` environment variable). In Cloudera ML, the application URL is:

```
https://<application-subdomain>.<workbench-domain>
```

| Environment | Authentication |
|---|---|
| Cloudera ML | Set `Authorization: Bearer {CDSW_APIV2_KEY}` header |
| Local development | No authentication required |

```python
import requests

# Local
BASE_URL = "http://localhost:8100"
headers = {}

# Cloudera ML
BASE_URL = "https://<application-subdomain>.<workbench-domain>"
headers = {"Authorization": f"Bearer {CDSW_APIV2_KEY}"}
```

---

## Health Check

Verify the service is running before issuing workflow requests.

```python
resp = requests.get(f"{BASE_URL}/health")
# {"status": "healthy", ...}
```

---

## Minimal Client Pattern -- Preview Mode

The three-step workflow in preview mode (`is_demo: true`) executes synchronously and returns inline results. Preview mode is limited to **25 items**.

### Step 1: Generate

POST to `/synthesis/generate` with a `SynthesisRequest` body.

```python
import requests

BASE_URL = "http://localhost:8100"

# Step 1: Generate synthetic data
gen_payload = {
    "use_case": "code_generation",
    "model_id": "us.anthropic.claude-3-5-haiku-20241022-v1:0",
    "inference_type": "aws_bedrock",
    "num_questions": 3,
    "technique": "sft",
    "topics": ["python_basics", "data_structures"],
    "is_demo": True,
    "max_concurrent_topics": 5,
    "examples": [
        {
            "question": "How do you create a list in Python?",
            "solution": "Use square brackets: my_list = [1, 2, 3]"
        }
    ],
    "model_params": {
        "temperature": 0.0,
        "top_p": 1.0,
        "top_k": 250,
        "max_tokens": 4096
    }
}

resp = requests.post(f"{BASE_URL}/synthesis/generate", json=gen_payload)
gen_result = resp.json()
file_path = gen_result["file_path"]
```

### Step 2: Evaluate

POST to `/synthesis/evaluate` with an `EvaluationRequest` body. Use the `file_path` returned from generation.

```python
# Step 2: Evaluate generated data
eval_payload = {
    "use_case": "code_generation",
    "model_id": "us.anthropic.claude-3-5-sonnet-20241022-v2:0",
    "inference_type": "aws_bedrock",
    "import_path": file_path,
    "import_type": "local",
    "is_demo": True,
    "export_type": "local",
    "max_workers": 4,
    "examples": [
        {"score": 4, "justification": "Well-structured code with good practices."},
        {"score": 2, "justification": "Basic implementation with issues."}
    ],
    "model_params": {
        "temperature": 0.0,
        "top_p": 1.0,
        "top_k": 250,
        "max_tokens": 4096
    }
}

resp = requests.post(f"{BASE_URL}/synthesis/evaluate", json=eval_payload)
eval_result = resp.json()
```

### Step 3: Export

POST to `/export_results` with an `Export\_synth` body.

```python
# Step 3: Export to HuggingFace
export_payload = {
    "export_type": ["huggingface"],
    "file_path": file_path,
    "output_key": "Prompt",
    "output_value": "Completion",
    "hf_config": {
        "hf_token": "hf_...",
        "hf_username": "your-username",
        "hf_repo_name": "my-synthetic-dataset",
        "hf_commit_message": "Initial export"
    }
}

resp = requests.post(f"{BASE_URL}/export_results", json=export_payload)
export_result = resp.json()
```

---

## Freeform Generation

The `/synthesis/freeform` endpoint uses `CustomPromptRequest` (not `SynthesisRequest`). The `custom\_p` field defaults to `true`.

```python
freeform_payload = {
    "model_id": "us.anthropic.claude-3-5-haiku-20241022-v1:0",
    "inference_type": "aws_bedrock",
    "custom_prompt": "Generate customer support conversations with fields: category, question, response, sentiment",
    "example": [
        {"category": "billing", "question": "How do I update payment?", "response": "Go to Settings > Billing", "sentiment": "neutral"}
    ]
}

resp = requests.post(f"{BASE_URL}/synthesis/freeform", json=freeform_payload)
```

Key differences from `/synthesis/generate`:

| | `/synthesis/generate` | `/synthesis/freeform` |
|---|---|---|
| Request model | `SynthesisRequest` | `CustomPromptRequest` |
| Few-shot field | `examples` (list of `{question, solution}`) | `example` (list of arbitrary dicts) |
| Prompt control | `custom\_prompt` optional override | `custom\_prompt` required |
| NaN handling | Standard JSON | Post-processed by `deep\_sanitize\_nans()` |

---

## Batch Mode

When `is_demo` is `false`, the API submits a CML job and returns a job ID instead of inline results. The client must poll the history endpoint until the job completes.

```python
# Submit batch job
gen_payload["is_demo"] = False
resp = requests.post(f"{BASE_URL}/synthesis/generate", json=gen_payload)
# Response: {"job_id": "...", "status": "running"}

# Poll for completion
import time
while True:
    history = requests.get(f"{BASE_URL}/synthesis/history?page=1&per_page=5").json()
    # Check if your job appears with status "completed"
    time.sleep(10)
```

Batch mode requires a Cloudera ML environment with CML Jobs API access.

| Mode | `is\_demo` | Execution | Response |
|---|---|---|---|
| Preview | `true` | Synchronous, inline | Direct JSON with generated data |
| Batch | `false` | Asynchronous via CML Jobs | `{"job\_id": "...", "status": "running"}` |

---

## Error Handling

All error responses use the standard envelope:

```json
{"status": "failed", "error": "..."}
```

Pydantic validation errors return `422` with FastAPI's default `detail` array.

```python
resp = requests.post(f"{BASE_URL}/synthesis/generate", json=payload)
if resp.status_code != 200:
    error = resp.json()
    # {"status": "failed", "error": "..."}
    print(f"Error {resp.status_code}: {error.get('error', 'Unknown')}")
```

| Status | Meaning | Common Cause |
|---|---|---|
| 400 | Bad Request | Missing required fields, invalid input |
| 404 | Not Found | File or resource not found |
| 422 | Validation Error | Pydantic schema validation failure |
| 500 | Internal Server Error | Provider error, unexpected failure |
| 503 | Service Unavailable | CAII endpoint downscaled |
| 504 | Gateway Timeout | Request exceeded endpoint timeout |

---

## Tips

- Use a different (typically stronger) model for evaluation than for generation.
- Set `temperature: 0.0` for evaluation to get deterministic scoring.
- The `max\_concurrent\_topics` parameter (generation) and `max\_workers` parameter (evaluation) control parallelism -- higher values increase throughput but also API rate pressure.
- Preview mode is limited to 25 items; use batch mode for larger datasets.
- The `custom\_prompt` field lets you override default prompts for both generation and evaluation.
