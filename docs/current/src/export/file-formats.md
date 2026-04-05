# Output File Formats

All pipeline outputs are written as JSON files to the local working directory.

## Generation Output Files

### SFT / Custom\_Workflow

Filename pattern: `qa_pairs_{model}_{timestamp}_{suffix}.json` where suffix is `test` (preview) or `final` (batch).

The file contains a JSON array of objects. When topics are used:

| Field | Type | Description |
|---|---|---|
| Seeds | string | The topic that generated this pair |
| {output\_key} | string | The generated question (default column name `Prompt`) |
| {output\_value} | string | The generated answer (default column name `Completion`) |

When `doc_paths` are used, `Seeds` is replaced by `Generated_From` containing the source document chunk.

```json
[
  {
    "Seeds": "python_basics",
    "Prompt": "How do you read a CSV file in Python?",
    "Completion": "Use pandas: pd.read_csv('file.csv')"
  }
]
```

Custom\_Workflow with `input_path` omits the `Seeds`/`Generated_From` column and writes only the `output_key` and `output_value` columns.

Source: `app/services/synthesis_legacy_service.py`

### Freeform

Filename pattern: `freeform_data_{model}_{timestamp}_{suffix}.json` where suffix is `test` (preview) or `final` (batch).

Each item has dynamic columns defined by the user's custom prompt schema. The LLM response is parsed into key-value pairs based on the freeform template. A provenance column is prepended:

| Field | Type | Description |
|---|---|---|
| Seeds | string | The topic used for generation (topic-based) |
| Generated\_From | string | The source document chunk (document-based) |
| *(dynamic)* | string | Columns extracted from the LLM response per the schema |

The dynamic columns typically include the `output_key` (default `Prompt`) and `output_value` (default `Completion`), plus any additional columns defined in the freeform template.

```json
[
  {
    "Seeds": "customer_support",
    "Prompt": "How do I reset my password?",
    "Completion": "Navigate to Settings > Security > Reset Password."
  }
]
```

Source: `app/services/synthesis_service.py`

## Evaluation Output Files

### SFT / Custom\_Workflow Evaluation

Filename pattern: `qa_pairs_{model}_{timestamp}_evaluated.json`

The file contains a JSON object keyed by topic, with an `Overall_Average` score at the top level.

| Field | Type | Description |
|---|---|---|
| {topic} | object | Per-topic evaluation statistics and results |
| {topic}.average\_score | float | Mean score for this topic |
| {topic}.min\_score | float | Lowest score in this topic |
| {topic}.max\_score | float | Highest score in this topic |
| {topic}.evaluated\_pairs | array | List of evaluated pair objects |
| {topic}.evaluated\_pairs[].question | string | The original question |
| {topic}.evaluated\_pairs[].solution | string | The original answer |
| {topic}.evaluated\_pairs[].evaluation.score | float | Numeric score (typically 1-5 scale) |
| {topic}.evaluated\_pairs[].evaluation.justification | string | Text explanation of the score |
| Overall\_Average | float | Mean score across all topics |

```json
{
  "python_basics": {
    "average_score": 4.25,
    "min_score": 3,
    "max_score": 5,
    "evaluated_pairs": [
      {
        "question": "How do you read a CSV file?",
        "solution": "Use pandas: pd.read_csv('file.csv')",
        "evaluation": {
          "score": 4,
          "justification": "Correct and concise answer."
        }
      }
    ],
    "total_evaluated": 10,
    "total_failed": 0
  },
  "Overall_Average": 4.25
}
```

Source: `app/services/evaluator_legacy_service.py`

### Freeform Evaluation

Filename pattern: `row_data_{model}_{timestamp}_evaluated.json`

The file contains a JSON object with statistics and an array of evaluated rows.

| Field | Type | Description |
|---|---|---|
| average\_score | float | Mean score across all rows |
| min\_score | float | Lowest score |
| max\_score | float | Highest score |
| evaluated\_rows | array | List of evaluated row objects |
| evaluated\_rows[].row | object | The original row data (all columns preserved) |
| evaluated\_rows[].evaluation.score | float | Numeric score (typically 1-5 scale) |
| evaluated\_rows[].evaluation.justification | string | Text explanation of the score |
| Overall\_Average | float | Mean score across all rows |

```json
{
  "average_score": 4.0,
  "min_score": 3,
  "max_score": 5,
  "evaluated_rows": [
    {
      "row": {
        "Seeds": "customer_support",
        "Prompt": "How do I reset my password?",
        "Completion": "Navigate to Settings > Security > Reset Password."
      },
      "evaluation": {
        "score": 4,
        "justification": "Clear and actionable response."
      }
    }
  ],
  "total_evaluated": 10,
  "total_failed": 0,
  "Overall_Average": 4.0
}
```

Source: `app/services/evaluator_service.py`

## Alignment Output

The Model Alignment pipeline returns DPO and KTO formatted data as an API response (not written to a file by default).

### DPO Format

Each object represents a preference pair derived from comparing the original and alternate completions.

| Field | Type | Description |
|---|---|---|
| instruction | string | The original question/prompt |
| chosen\_response | string | The higher-scored completion |
| rejected\_response | string | The lower-scored completion |
| chosen\_rating | float | Score of the chosen completion |
| rejected\_rating | float | Score of the rejected completion |

```json
[
  {
    "instruction": "Explain recursion.",
    "chosen_response": "Recursion is when a function calls itself...",
    "rejected_response": "It is a loop.",
    "chosen_rating": 5,
    "rejected_rating": 2
  }
]
```

### KTO Format

Each DPO pair yields two KTO entries: one labeled `true` (chosen) and one labeled `false` (rejected).

| Field | Type | Description |
|---|---|---|
| Prompt | string | The original question/prompt |
| Completion | string | One of the two completions |
| Label | bool | `true` if this was the higher-scored completion, `false` otherwise |
| Rating | float | The evaluation score for this completion |

```json
[
  {
    "Prompt": "Explain recursion.",
    "Completion": "Recursion is when a function calls itself...",
    "Label": true,
    "Rating": 5
  },
  {
    "Prompt": "Explain recursion.",
    "Completion": "It is a loop.",
    "Label": false,
    "Rating": 2
  }
]
```

Source: `app/services/model_alignment.py`

## Column Mapping

The `output_key` and `output_value` fields on `SynthesisRequest` and `Export_synth` control column naming throughout the pipeline.

| Parameter | Default | Maps To |
|---|---|---|
| output\_key | `"Prompt"` | The input/question column |
| output\_value | `"Completion"` | The output/answer column |

For HuggingFace export, the Dataset features are constructed using these column names plus either a `Seeds` column (topic-based generation) or a `Generated_From` column (document-based generation), determined by whether `doc_paths` was set on the original generation request.

Source: `app/models/request_models.py`, `app/services/export_results.py`

## File Size Limits

The system enforces a 10 GB total dataset size limit, checked before generation starts. Files are validated via the `JsonDataSize` model with `input_path` (list of file paths) and `input_key` (column to measure). The size check sums the byte sizes of all input files and returns HTTP 413 if the limit is exceeded.

| Parameter | Type | Default | Description |
|---|---|---|---|
| input\_path | list[str] | -- | File paths to measure |
| input\_key | str | `"Prompt"` | Column name used for size calculation |

Source: `app/models/request_models.py`, `app/main.py`
