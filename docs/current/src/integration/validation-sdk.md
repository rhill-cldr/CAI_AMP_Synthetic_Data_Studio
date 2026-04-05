# Building a Validation SDK

This chapter covers programmatic validation of SDS output artifacts. It provides schema definitions, quality checks, and metadata verification patterns for engineers building tooling around SDS-generated data.

## Output Schema Validation

All SDS outputs are JSON arrays. The schema of each element depends on the generation technique.

### Schema by Technique

| Technique | Fields | Notes |
|-----------|--------|-------|
| SFT / Custom\_Workflow | `Seeds`, `Prompt`, `Completion` | `Seeds` replaced by `Generated\_From` when `doc\_paths` was used |
| Freeform | Dynamic columns from custom prompt | Always includes `output\_key` (default `Prompt`) and `output\_value` (default `Completion`) |
| Evaluation | Same as generation output + `score`, `justification` | `score` is a float; `justification` is a string |
| DPO | `prompt`, `chosen`, `rejected` | All lowercase field names |
| KTO | `prompt`, `completion`, `label` | `label` is a boolean |

### SFT / Custom\_Workflow Output

```json
[
  {
    "Seeds": "python_basics",
    "Prompt": "How do you create a list?",
    "Completion": "Use square brackets..."
  }
]
```

When `doc_paths` was provided in the generation request, the `Seeds` field is replaced by `Generated_From`:

```json
[
  {
    "Generated_From": "architecture_guide.pdf",
    "Prompt": "What are the main components?",
    "Completion": "The system consists of..."
  }
]
```

### Freeform Output

Freeform generation uses dynamic columns defined by the custom prompt. The column names are controlled by `output_key` (default `Prompt`) and `output_value` (default `Completion`) in the request.

### Evaluation Output

Evaluation appends `score` and `justification` to each record from the source generation file:

```json
[
  {
    "Seeds": "python_basics",
    "Prompt": "How do you create a list?",
    "Completion": "Use square brackets...",
    "score": 4.0,
    "justification": "The code is well-structured, includes error handling, and follows Python best practices."
  }
]
```

### DPO Output

```json
[
  {
    "prompt": "Explain closures in Python.",
    "chosen": "A closure is a function that retains access to...",
    "rejected": "Closures are just nested functions."
  }
]
```

### KTO Output

```json
[
  {
    "prompt": "How do you handle exceptions?",
    "completion": "Use try/except blocks...",
    "label": true
  }
]
```

## Pydantic Validation Models

Define Pydantic models that mirror SDS output schemas. These models provide type checking, constraint validation, and clear error messages.

```python
from pydantic import BaseModel, Field, field_validator
from typing import List, Optional


class SFTRecord(BaseModel):
    Seeds: str
    Prompt: str
    Completion: str


class DocRecord(BaseModel):
    Generated_From: str
    Prompt: str
    Completion: str


class EvaluatedRecord(SFTRecord):
    score: float = Field(ge=0, le=5)
    justification: str


class DPORecord(BaseModel):
    prompt: str
    chosen: str
    rejected: str


class KTORecord(BaseModel):
    prompt: str
    completion: str
    label: bool


def validate_sft_output(data: list) -> List[SFTRecord]:
    """Validate a list of SFT generation records."""
    return [SFTRecord(**item) for item in data]


def validate_evaluated_output(data: list) -> List[EvaluatedRecord]:
    """Validate a list of evaluated SFT records."""
    return [EvaluatedRecord(**item) for item in data]


def validate_dpo_output(data: list) -> List[DPORecord]:
    """Validate a list of DPO alignment records."""
    return [DPORecord(**item) for item in data]


def validate_kto_output(data: list) -> List[KTORecord]:
    """Validate a list of KTO alignment records."""
    return [KTORecord(**item) for item in data]
```

Choosing the correct model depends on the technique. A dispatcher pattern simplifies this:

```python
VALIDATORS = {
    "sft": SFTRecord,
    "custom_workflow": SFTRecord,
    "evaluation": EvaluatedRecord,
    "dpo": DPORecord,
    "kto": KTORecord,
}


def validate_output(data: list, technique: str) -> list:
    """Validate output records using the appropriate model for the technique."""
    model = VALIDATORS.get(technique)
    if model is None:
        raise ValueError(f"Unknown technique: {technique}")
    return [model(**item) for item in data]
```

## Quality Checks

### Load and Inspect

```python
import json
from pathlib import Path


def load_and_validate(file_path: str) -> dict:
    """Load a JSON output file and run basic quality checks."""
    path = Path(file_path)

    if not path.exists():
        raise FileNotFoundError(f"Output file not found: {file_path}")

    with open(path) as f:
        data = json.load(f)

    if not isinstance(data, list):
        raise ValueError("Expected JSON array")

    if len(data) == 0:
        raise ValueError("Empty dataset")

    return {
        "record_count": len(data),
        "columns": list(data[0].keys()),
        "sample": data[0],
    }
```

### Score Distribution Check

Use this for evaluated data to verify scores fall within the expected range and to surface distribution statistics.

```python
def check_score_distribution(data: list, min_score=1, max_score=5):
    """Verify scores fall within expected range and check distribution."""
    scores = [item["score"] for item in data]

    out_of_range = [s for s in scores if s < min_score or s > max_score]
    if out_of_range:
        raise ValueError(f"Scores out of range: {out_of_range}")

    avg = sum(scores) / len(scores)
    return {
        "count": len(scores),
        "min": min(scores),
        "max": max(scores),
        "mean": round(avg, 2),
        "out_of_range": len(out_of_range),
    }
```

### Completeness Check

Verify that every record contains the required columns with non-empty values.

```python
def check_completeness(data: list, required_columns: list):
    """Verify all records have the required columns with non-empty values."""
    issues = []
    for i, record in enumerate(data):
        for col in required_columns:
            if col not in record:
                issues.append(f"Record {i}: missing column '{col}'")
            elif not record[col] or (
                isinstance(record[col], str) and not record[col].strip()
            ):
                issues.append(f"Record {i}: empty value for '{col}'")
    return issues
```

### Required Columns by Technique

| Technique | Required Columns |
|-----------|-----------------|
| SFT / Custom\_Workflow | `Seeds`, `Prompt`, `Completion` |
| SFT with doc\_paths | `Generated\_From`, `Prompt`, `Completion` |
| Evaluation | `Seeds`, `Prompt`, `Completion`, `score`, `justification` |
| DPO | `prompt`, `chosen`, `rejected` |
| KTO | `prompt`, `completion`, `label` |

## Database Metadata Validation

SDS stores metadata for every generation and evaluation job in SQLite (via the `generation_metadata` and `evaluation_metadata` tables). Use the history API to cross-reference output files with their metadata.

### History API Endpoint

| Method | Path | Parameters |
|--------|------|------------|
| GET | `/generations/history` | `page` (int, default 1), `page\_size` (int, default 10, max 100) |

The response includes a `data` array and a `pagination` object:

```json
{
  "data": [
    {
      "id": 1,
      "timestamp": "2024-12-04T13:24:11",
      "display_name": "my_dataset",
      "generate_file_name": "qa_pairs_claude_20241204_132411_test.json",
      "model_id": "us.anthropic.claude-3-5-haiku-20241022-v1:0",
      "num_questions": 10,
      "total_count": 30,
      "use_case": "code_generation",
      "job_status": "ENGINE_SUCCEEDED",
      "local_export_path": "/data/qa_pairs_claude_20241204_132411_test.json",
      "hf_export_path": null,
      "completed_rows": 30
    }
  ],
  "pagination": {
    "total": 1,
    "page": 1,
    "page_size": 10,
    "total_pages": 1
  }
}
```

### Verify Generation Metadata

```python
import requests


def verify_generation_metadata(base_url: str, filename: str):
    """Check that a generation file has corresponding metadata."""
    resp = requests.get(
        f"{base_url}/generations/history",
        params={"page": 1, "page_size": 100},
    )
    resp.raise_for_status()
    history = resp.json()

    matching = [
        r
        for r in history.get("data", [])
        if r.get("generate_file_name") == filename
    ]
    if not matching:
        raise ValueError(f"No metadata found for {filename}")

    record = matching[0]
    return {
        "model_id": record.get("model_id"),
        "use_case": record.get("use_case"),
        "num_questions": record.get("num_questions"),
        "total_count": record.get("total_count"),
        "job_status": record.get("job_status"),
        "timestamp": record.get("timestamp"),
    }
```

### Fetch Full Metadata by Filename

The `/generations/{file_name}` endpoint returns the complete metadata record, including model parameters, topics, examples, and document paths:

```python
def get_full_metadata(base_url: str, filename: str) -> dict:
    """Retrieve the full generation metadata for a specific file."""
    resp = requests.get(f"{base_url}/generations/{filename}")
    if resp.status_code == 404:
        raise ValueError(f"No generation found with filename: {filename}")
    resp.raise_for_status()
    return resp.json()
```

## Integration with Testing Frameworks

The following pytest examples validate SDS output files as part of a CI pipeline.

### SFT Output Validation

```python
import pytest
import json


@pytest.fixture
def sft_output():
    with open("qa_pairs_claude_20241204_132411_test.json") as f:
        return json.load(f)


def test_sft_output_is_list(sft_output):
    assert isinstance(sft_output, list)
    assert len(sft_output) > 0


def test_sft_records_have_required_fields(sft_output):
    required = {"Seeds", "Prompt", "Completion"}
    for record in sft_output:
        assert required.issubset(record.keys()), (
            f"Missing fields: {required - record.keys()}"
        )


def test_no_empty_completions(sft_output):
    for record in sft_output:
        assert record["Completion"].strip(), "Empty completion found"
```

### Evaluation Output Validation

```python
@pytest.fixture
def eval_output():
    with open("evaluated_qa_pairs_claude_20241204_140000.json") as f:
        return json.load(f)


def test_eval_records_have_score_and_justification(eval_output):
    for record in eval_output:
        assert "score" in record, "Missing score field"
        assert "justification" in record, "Missing justification field"
        assert isinstance(record["score"], (int, float))
        assert 0 <= record["score"] <= 5, f"Score out of range: {record['score']}"


def test_eval_justifications_are_nonempty(eval_output):
    for record in eval_output:
        assert record["justification"].strip(), "Empty justification found"
```

### DPO Output Validation

```python
@pytest.fixture
def dpo_output():
    with open("dpo_alignment_output.json") as f:
        return json.load(f)


def test_dpo_records_have_required_fields(dpo_output):
    required = {"prompt", "chosen", "rejected"}
    for record in dpo_output:
        assert required.issubset(record.keys()), (
            f"Missing fields: {required - record.keys()}"
        )


def test_dpo_chosen_differs_from_rejected(dpo_output):
    for record in dpo_output:
        assert record["chosen"] != record["rejected"], (
            "Chosen and rejected responses are identical"
        )
```

## Source Files

| File | Purpose |
|------|---------|
| `app/models/request\_models.py` | Request/response Pydantic models, `Technique` enum, `Export\_synth` schema |
| `app/services/export\_results.py` | Export service, HuggingFace dataset creation with `Seeds`/`Generated\_From` logic |
| `app/core/database.py` | `DatabaseManager` class, `generation\_metadata` and `evaluation\_metadata` table definitions |
