# Generation Routes

Four endpoints handle synthetic data generation: two produce data (`/synthesis/generate` and `/synthesis/freeform`), one builds an AI-enhanced prompt (`/create_custom_prompt`), and one previews the assembled LLM prompt (`/complete_gen_prompt`).

All generation endpoints accept JSON request bodies and return JSON responses.

---

## POST /synthesis/generate

Generate question-answer pairs using legacy techniques (SFT, Custom\_Workflow).

### Request Body -- `SynthesisRequest`

| Field | Type | Default | Description |
|---|---|---|---|
| use\_case | `UseCase` enum | `"custom"` | Domain config: `code_generation`, `text2sql`, `custom`, `lending_data`, `credit_card_data`, `ticketing_dataset` |
| model\_id | string | **required** | LLM model identifier (e.g., `"us.anthropic.claude-3-5-haiku-20241022-v1:0"`) |
| num\_questions | int | `1` | Number of items to generate (gt=0) |
| technique | `Technique` enum | `"sft"` | `"sft"`, `"custom_workflow"`, `"model_alignment"`, `"freeform"` |
| is\_demo | bool | `true` | `true` = preview mode (sync), `false` = batch mode (CML job) |
| inference\_type | string | `"aws_bedrock"` | Provider: `aws_bedrock`, `CAII`, `openai`, `openai_compatible`, `gemini` |
| caii\_endpoint | string? | `null` | CAII model serving endpoint URL |
| openai\_compatible\_endpoint | string? | `null` | OpenAI-compatible endpoint URL |
| topics | List\[str\]? | `null` | Topics for SFT generation |
| doc\_paths | List\[str\]? | `null` | Document paths for context |
| input\_path | List\[str\]? | `null` | Seed input file paths (Custom\_Workflow) |
| input\_key | string? | `"Prompt"` | Key name in seed data for input column |
| output\_key | string? | `"Prompt"` | Key name for output column |
| output\_value | string? | `"Completion"` | Value name for output column |
| examples | List\[Example\]? | `null` | Few-shot examples `[{question, solution}]` |
| example\_custom | List\[Dict\]? | `null` | Freeform examples (flexible schema) |
| example\_path | string? | `null` | Path to examples file |
| schema | string? | `null` | SQL schema for `text2sql` use case |
| custom\_prompt | string? | `null` | Custom prompt instructions |
| display\_name | string? | `null` | Human-readable name for UI |
| max\_concurrent\_topics | int | `5` | Concurrent topic processing (1--100) |
| model\_params | `ModelParameters`? | `null` | Temperature, top\_p, min\_p, top\_k, max\_tokens |

### ModelParameters

| Field | Type | Default | Range |
|---|---|---|---|
| temperature | float | `0.0` | 0.0--2.0 |
| top\_p | float | `1.0` | 0.0--1.0 |
| min\_p | float | `0.0` | 0.0--1.0 |
| top\_k | int | `150` | >= 0 |
| max\_tokens | int | `8192` | >= 1 |

### Routing Logic

1. If `inference_type` is `"CAII"`, validates endpoint via `caii_check()`.
2. If `doc_paths` present and total size > 1 GB and <= 10 GB: forces batch mode, scales `memory = size + 2`, `cpu = max(2, size // 2)`.
3. If `doc_paths` total size > 10 GB: returns **413 Payload Too Large**.
4. **Preview mode** (`is_demo=true`): routes to `SynthesisLegacyService`. If `input_path` present, calls `generate_result()` (Custom\_Workflow). Otherwise calls `generate_examples()` (SFT).
5. **Batch mode** (`is_demo=false`): routes to `synthesis_job.generate_job()`.

### Response

- **Preview mode:** direct JSON with generated data.
- **Batch mode:** job metadata with `job_id` for status polling.

---

## POST /synthesis/freeform

Generate freeform structured data based on examples and custom instructions.

### Request Body

Same `SynthesisRequest` model as `/synthesis/generate`.

### Routing Logic

1. Same CAII validation and document size checks as `/synthesis/generate`.
2. **Preview mode:** routes to `SynthesisService.generate_freeform()`. Result passes through `deep_sanitize_nans()` and `jsonable_encoder()`.
3. **Batch mode:** routes to `synthesis_job.generate_job()` with `freeform=True`.

---

## POST /create\_custom\_prompt

Generate an AI-enhanced custom prompt from user instructions.

### Request Body -- `CustomPromptRequest`

| Field | Type | Default | Description |
|---|---|---|---|
| model\_id | string | **required** | LLM model identifier |
| custom\_prompt | string | **required** | User's prompt instructions |
| inference\_type | string | `"aws_bedrock"` | Provider |
| caii\_endpoint | string? | `null` | CAII endpoint |
| example\_path | string? | `null` | Path to examples file |
| example | List\[Dict\]? | `null` | Inline examples |
| custom\_p | bool | `true` | Custom prompt flag |

### Response

```json
{"generated_prompt": "..."}
```

---

## POST /complete\_gen\_prompt

Preview the fully-assembled prompt that will be sent to the LLM for generation.

### Request Body

Same `SynthesisRequest` model as `/synthesis/generate`.

### Behavior by Technique

| Technique | Method Called |
|---|---|
| Freeform | `PromptBuilder.build_freeform_prompt()` |
| Custom\_Workflow | Reads first item from `input_path`, calls `PromptBuilder.build_generate_result_prompt()` |
| SFT | `PromptBuilder.build_prompt()` |

### Response

```json
{"complete_prompt": "..."}
```

---

## Technique Routing Summary

| Technique | Route | Preview Service | Batch Service |
|---|---|---|---|
| SFT | `POST /synthesis/generate` | `SynthesisLegacyService.generate_examples()` | `synthesis_job.generate_job()` |
| Custom\_Workflow | `POST /synthesis/generate` | `SynthesisLegacyService.generate_result()` | `synthesis_job.generate_job()` |
| Freeform | `POST /synthesis/freeform` | `SynthesisService.generate_freeform()` | `synthesis_job.generate_job(freeform=True)` |
