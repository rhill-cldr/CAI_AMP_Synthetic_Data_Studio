# Evaluation Routes

Three endpoints handle evaluation of generated data: `/synthesis/evaluate` scores legacy QA pairs, `/synthesis/evaluate_freeform` scores freeform rows, and `/complete_eval_prompt` previews the assembled LLM evaluation prompt.

All evaluation endpoints accept JSON request bodies and return JSON responses.

---

## POST /synthesis/evaluate

Evaluate generated QA pairs (SFT and Custom\_Workflow techniques).

### Request Body -- `EvaluationRequest`

| Field | Type | Default | Description |
|---|---|---|---|
| use\_case | `UseCase` enum | **required** | Domain configuration: `code_generation`, `text2sql`, `custom`, `lending_data`, `credit_card_data`, `ticketing_dataset` |
| technique | `Technique` enum | `"sft"` | Generation technique used |
| model\_id | string | **required** | LLM model identifier for evaluation |
| import\_path | string? | `null` | Path to the generated data file to evaluate |
| import\_type | string | `"local"` | Source type for import |
| is\_demo | bool | `true` | `true` = preview mode (sync), `false` = batch mode (CML job) |
| inference\_type | string | `"aws_bedrock"` | Provider: `aws_bedrock`, `CAII`, `openai`, `openai_compatible`, `gemini` |
| caii\_endpoint | string? | `null` | CAII model serving endpoint URL |
| examples | List\[Example\_eval\]? | `null` | Few-shot scoring examples `[{score, justification}]` |
| custom\_prompt | string? | `null` | Custom evaluation rubric |
| display\_name | string? | `null` | Human-readable name for UI |
| output\_key | string? | `"Prompt"` | Key field name in data |
| output\_value | string? | `"Completion"` | Value field name in data |
| max\_workers | int | `4` | Parallel evaluation workers (1--100) |
| export\_type | string | `"local"` | Export destination: `local` or `s3` |
| s3\_config | `S3Config`? | `null` | S3 export configuration |
| model\_params | `ModelParameters`? | `null` | Temperature, top\_p, min\_p, top\_k, max\_tokens |

### Example\_eval

| Field | Type | Description |
|---|---|---|
| score | float | Numeric score |
| justification | string | Text explanation for the score |

### Routing Logic

1. If `inference_type` is `"CAII"`, validates endpoint via `caii_check()`.
2. **Preview mode** (`is_demo=true`): routes to `EvaluatorLegacyService.evaluate_results()`.
3. **Batch mode** (`is_demo=false`): routes to `synthesis_job.evaluate_job()`.

### Response

- **Preview mode:** direct JSON with scored data.
- **Batch mode:** job metadata with `job_id` for status polling.

---

## POST /synthesis/evaluate\_freeform

Evaluate freeform data rows.

### Request Body

Same `EvaluationRequest` model as `/synthesis/evaluate`.

### Routing Logic

1. Same CAII validation as `/synthesis/evaluate`.
2. **Preview mode** (`is_demo=true`): routes to `EvaluatorService.evaluate_row_data()`.
3. **Batch mode** (`is_demo=false`): routes to `synthesis_job.evaluate_job()` with `freeform=True`.

---

## POST /complete\_eval\_prompt

Preview the fully-assembled prompt that will be sent to the LLM for evaluation.

### Request Body

Same `EvaluationRequest` model as `/synthesis/evaluate`.

### Behavior by Technique

| Technique | Method Called |
|---|---|
| Freeform | Reads data from `import_path`, calls `PromptBuilder.build_freeform_eval_prompt()` with first row |
| SFT / Custom\_Workflow | Reads data from `import_path`, extracts QA pairs using `output_key`/`output_value`, calls `PromptBuilder.build_eval_prompt()` with first pair |

### Response

```json
{"complete_prompt": "..."}
```

---

## Scoring System

Each use case in `USE_CASE_CONFIGS_EVALS` defines a scoring rubric prompt and default few-shot examples with scores and justifications.

| Use Case | Score Range | Rubric Style |
|---|---|---|
| `code_generation` | 1--5 | Additive 5-point system (basic functionality through mastery) |
| `text2sql` | 1--5 | Additive 5-point system (basic data retrieval through optimization) |
| `custom` | 1--5 | Generic quality scale |
| `lending_data` | 1--10 | Deductive from 10 (privacy, formatting, cross-column, realism) |
| `credit_card_data` | 1--10 | Deductive from 10 (privacy, formatting, data series, cross-column) |
| `ticketing_dataset` | 1--5 | Quality of query + response keyword matching |

When `custom_prompt` is provided on the request, it overrides the default rubric. When `examples` is provided, those replace the default few-shot examples.

---

## Evaluation Routing Summary

| Technique | Route | Preview Service | Batch Service |
|---|---|---|---|
| SFT / Custom\_Workflow | `POST /synthesis/evaluate` | `EvaluatorLegacyService.evaluate_results()` | `synthesis_job.evaluate_job()` |
| Freeform | `POST /synthesis/evaluate_freeform` | `EvaluatorService.evaluate_row_data()` | `synthesis_job.evaluate_job(freeform=True)` |
