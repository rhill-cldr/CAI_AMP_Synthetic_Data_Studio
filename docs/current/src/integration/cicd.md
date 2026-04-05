# CI/CD Integration

SDS can be driven entirely via its REST API, making it suitable for CI/CD integration. Common automation patterns include:

- Scheduled dataset regeneration
- Automated quality validation on new datasets
- Export to HuggingFace Hub or S3 as part of a release pipeline
- Regression testing of prompt templates

---

## GitHub Actions Example

The following workflow starts the SDS backend, runs a generation request, validates the output, and uploads the result as a build artifact. It runs weekly on Monday at 06:00 UTC and can be triggered manually.

```yaml
name: Generate and Validate Dataset

on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly on Monday
  workflow_dispatch:

env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: us-west-2
  SDS_URL: http://localhost:8100

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: pip install uv && uv sync

      - name: Start SDS backend
        run: |
          uv run python app/run.py &
          sleep 10  # Wait for startup

      - name: Health check
        run: curl -f $SDS_URL/health

      - name: Generate dataset
        run: |
          python -c "
          import requests, json
          payload = {
              'use_case': 'code_generation',
              'model_id': 'us.anthropic.claude-3-5-haiku-20241022-v1:0',
              'inference_type': 'aws_bedrock',
              'num_questions': 5,
              'technique': 'sft',
              'topics': ['python_basics'],
              'is_demo': True,
              'model_params': {'temperature': 0.0, 'max_tokens': 4096}
          }
          resp = requests.post('$SDS_URL/synthesis/generate', json=payload)
          resp.raise_for_status()
          result = resp.json()
          print(json.dumps(result, indent=2))
          with open('gen_result.json', 'w') as f:
              json.dump(result, f)
          "

      - name: Validate output
        run: |
          python -c "
          import json
          with open('gen_result.json') as f:
              result = json.load(f)
          assert 'file_path' in result, 'Missing file_path'
          data = json.load(open(result['file_path']))
          assert len(data) > 0, 'Empty dataset'
          for record in data:
              assert 'Prompt' in record, 'Missing Prompt field'
              assert 'Completion' in record, 'Missing Completion field'
          print(f'Validated {len(data)} records')
          "

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: synthetic-dataset
          path: '*.json'
```

---

## Pipeline Script

A standalone Python script suitable for use in any CI/CD system. It waits for the SDS backend to become healthy, runs generation, and validates the output.

```python
#!/usr/bin/env python3
"""SDS CI/CD pipeline script."""
import requests
import json
import sys
import time

SDS_URL = "http://localhost:8100"


def wait_for_health(timeout=60):
    """Wait for SDS to become healthy."""
    start = time.time()
    while time.time() - start < timeout:
        try:
            resp = requests.get(f"{SDS_URL}/health", timeout=5)
            if resp.status_code == 200:
                return True
        except requests.ConnectionError:
            pass
        time.sleep(2)
    raise TimeoutError("SDS did not become healthy")


def generate(config: dict) -> dict:
    """Run generation and return result."""
    resp = requests.post(f"{SDS_URL}/synthesis/generate", json=config)
    resp.raise_for_status()
    return resp.json()


def evaluate(config: dict) -> dict:
    """Run evaluation and return result."""
    resp = requests.post(f"{SDS_URL}/synthesis/evaluate", json=config)
    resp.raise_for_status()
    return resp.json()


def validate_output(file_path: str, min_records: int = 1):
    """Validate generated output meets minimum requirements."""
    with open(file_path) as f:
        data = json.load(f)

    assert isinstance(data, list), "Output must be a JSON array"
    assert len(data) >= min_records, f"Expected >= {min_records} records, got {len(data)}"

    for i, record in enumerate(data):
        assert "Prompt" in record, f"Record {i}: missing Prompt"
        assert "Completion" in record, f"Record {i}: missing Completion"
        assert record["Prompt"].strip(), f"Record {i}: empty Prompt"
        assert record["Completion"].strip(), f"Record {i}: empty Completion"

    return len(data)


if __name__ == "__main__":
    wait_for_health()

    result = generate({
        "use_case": "code_generation",
        "model_id": "us.anthropic.claude-3-5-haiku-20241022-v1:0",
        "inference_type": "aws_bedrock",
        "num_questions": 5,
        "technique": "sft",
        "topics": ["python_basics"],
        "is_demo": True
    })

    count = validate_output(result["file_path"])
    print(f"Pipeline complete: {count} records validated")
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| AWS\_ACCESS\_KEY\_ID | For Bedrock | AWS access key |
| AWS\_SECRET\_ACCESS\_KEY | For Bedrock | AWS secret key |
| AWS\_DEFAULT\_REGION | For Bedrock | AWS region |
| CDP\_TOKEN | For CAII | Cloudera AI Inference token |
| OPENAI\_API\_KEY | For OpenAI | OpenAI API key |
| CDSW\_APP\_PORT | No | Override default port 8100 |
| CDSW\_APIV2\_KEY | CML only | CML application API key |

---

## Best Practices

- **Use preview mode for CI validation** -- set `is_demo: true` and keep `num_questions` small (3-5) to minimize LLM costs and execution time.
- **Pin model IDs** -- use explicit model IDs rather than aliases to ensure reproducibility.
- **Set temperature to 0** -- deterministic output makes test assertions reliable.
- **Validate before export** -- always check output structure and quality scores before pushing to HuggingFace or S3.
- **Health check first** -- always verify the SDS backend is healthy before submitting requests.
- **Timeout handling** -- generation requests can take minutes for large datasets; set appropriate HTTP timeouts.

---

## Timeout Reference

Per-endpoint server timeouts are enforced by the global middleware. Set your HTTP client timeout to at least the server timeout plus a buffer.

| Endpoint Pattern | Server Timeout |
|---|---|
| `*/generate` | 200s |
| `*/freeform` | 300s |
| `*/evaluate` | 200s |
| `*/export_results` | 200s |
| `*health*` | 5s |
| Default | 60s |

Source: `app/main.py` -- `get_timeout_for_request()`. See [API Conventions](../api/conventions.md) for the full timeout table.
