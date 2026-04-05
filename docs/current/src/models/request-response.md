# Request & Response Schemas

All request and response models are defined as Pydantic `BaseModel` subclasses. Validation constraints (ranges, required fields, enum membership) are enforced at deserialisation time by FastAPI and Pydantic before any route handler executes.

Source: `app/models/request_models.py`, `app/core/config.py`.

---

## SynthesisRequest

The primary request body for both `/synthesis/generate` and `/synthesis/freeform`. Every generation endpoint deserialises the incoming JSON into this model.

| Field | Type | Default | Validation | Description |
|---|---|---|---|---|
| `use_case` | `UseCase \| None` | `UseCase.CUSTOM` | enum | Domain-specific configuration (see [UseCase](#usecase)) |
| `model_id` | `str` | **required** | -- | LLM model identifier (e.g., `"us.anthropic.claude-3-5-haiku-20241022-v1:0"`) |
| `num_questions` | `int \| None` | `1` | `gt=0` | Number of items to generate per topic |
| `technique` | `Technique \| None` | `Technique.SFT` | enum | Generation method (see [Technique](#technique)) |
| `is_demo` | `bool` | `True` | -- | `true` = preview mode (sync), `false` = batch mode (CML job) |
| `inference_type` | `str \| None` | `"aws_bedrock"` | -- | LLM provider backend: `aws_bedrock`, `CAII`, `openai`, `openai_compatible`, `gemini` |
| `caii_endpoint` | `str \| None` | `None` | -- | CAII model serving endpoint URL |
| `openai_compatible_endpoint` | `str \| None` | `None` | -- | OpenAI-compatible endpoint URL |
| `topics` | `List[str] \| None` | `None` | -- | Seed topics for generation (SFT, Freeform) |
| `doc_paths` | `List[str] \| None` | `None` | -- | Document paths for chunk-based context injection |
| `input_path` | `List[str] \| None` | `None` | -- | Input file paths for seed data (Custom\_Workflow, Model\_Alignment) |
| `input_key` | `str \| None` | `"Prompt"` | -- | Column name to read from input files |
| `output_key` | `str \| None` | `"Prompt"` | -- | Column name for prompts in generated output |
| `output_value` | `str \| None` | `"Completion"` | -- | Column name for completions in generated output |
| `examples` | `List[Example] \| None` | `None` | -- | Few-shot examples for SFT `[{question, solution}]` |
| `example_custom` | `List[Dict[str, Any]] \| None` | `None` | -- | Flexible-schema examples for Freeform generation |
| `example_path` | `str \| None` | `None` | -- | Path to an external examples file |
| `schema` | `str \| None` | `None` | -- | SQL schema definition (text2sql use case) |
| `custom_prompt` | `str \| None` | `None` | -- | Override the default system prompt |
| `display_name` | `str \| None` | `None` | -- | Human-readable label for the UI and history |
| `max_concurrent_topics` | `int \| None` | `5` | `ge=1, le=100` | Maximum number of concurrent topics processed via ThreadPool |
| `model_params` | `ModelParameters \| None` | `None` | -- | Low-level model generation parameters |

### Technique Routing

The `technique` field determines which service handles the request and which fields are meaningful. Fields not listed for a technique are ignored.

| Field | SFT | Custom\_Workflow | Freeform | Model\_Alignment |
|---|---|---|---|---|
| `topics` | Required | -- | Required | -- |
| `input_path` | -- | Required | -- | Required |
| `examples` | Optional | Optional | -- | -- |
| `example_custom` | -- | -- | Optional | -- |
| `doc_paths` | Optional | -- | Optional | -- |
| `schema` | text2sql only | text2sql only | text2sql only | -- |

---

## SynthesisResponse

Returned by generation endpoints in preview mode.

| Field | Type | Default | Description |
|---|---|---|---|
| `job_id` | `str` | **required** | Unique identifier for the generation job |
| `status` | `str` | **required** | Job status (e.g., `"completed"`, `"failed"`) |
| `topics_processed` | `List[str]` | **required** | List of topics that were processed |
| `qa_pairs` | `Dict[str, List[Example]]` | **required** | Topic-to-QA-pairs mapping |
| `export_path` | `str \| None` | `None` | Path to the exported output file |
| `error` | `str \| None` | `None` | Error message if generation failed |

---

## EvaluationRequest

The request body for `/synthesis/evaluate` and `/synthesis/evaluate_freeform`.

| Field | Type | Default | Validation | Description |
|---|---|---|---|---|
| `use_case` | `UseCase` | **required** | enum | Domain configuration |
| `technique` | `Technique \| None` | `Technique.SFT` | enum | Technique of the data being evaluated |
| `model_id` | `str` | **required** | -- | LLM model identifier for evaluation |
| `import_path` | `str \| None` | `None` | -- | Path to the generated data file to evaluate |
| `import_type` | `str` | `"local"` | -- | Source type for import |
| `is_demo` | `bool` | `True` | -- | `true` = preview mode (sync), `false` = batch mode (CML job) |
| `inference_type` | `str \| None` | `"aws_bedrock"` | -- | Provider: `aws_bedrock`, `CAII`, `openai`, `openai_compatible`, `gemini` |
| `caii_endpoint` | `str \| None` | `None` | -- | CAII model serving endpoint URL |
| `examples` | `List[Example_eval] \| None` | `None` | -- | Few-shot scoring examples `[{score, justification}]` |
| `custom_prompt` | `str \| None` | `None` | -- | Custom evaluation rubric (overrides default) |
| `display_name` | `str \| None` | `None` | -- | Human-readable name for UI and history |
| `output_key` | `str \| None` | `"Prompt"` | -- | Key field name in data |
| `output_value` | `str \| None` | `"Completion"` | -- | Value field name in data |
| `max_workers` | `int \| None` | `4` | `ge=1, le=100` | Parallel evaluation worker threads |
| `export_type` | `str` | `"local"` | -- | Export destination: `"local"` or `"s3"` |
| `s3_config` | `S3Config \| None` | `None` | -- | S3 export configuration (when `export_type="s3"`) |
| `model_params` | `ModelParameters \| None` | `None` | -- | Low-level model generation parameters |

---

## Supporting Models

### Technique

Source: `app/models/request_models.py` -- `Technique(str, Enum)`.

The `Technique` enum controls service routing and prompt construction.

| Value | Description |
|---|---|
| `sft` | Supervised fine-tuning -- topic-to-QA-pair generation |
| `custom_workflow` | Seed-input-to-completion pipeline |
| `model_alignment` | Composite pipeline producing DPO/KTO preference data |
| `freeform` | Open-ended structured output with flexible schema |

### UseCase

Source: `app/core/config.py` -- `UseCase(str, Enum)`.

| Value | Description |
|---|---|
| `code_generation` | Programming Q&A pairs |
| `text2sql` | Natural language to SQL |
| `custom` | Blank template for user-defined tasks |
| `lending_data` | LendingClub-style loan records |
| `credit_card_data` | Credit card applicant profiles with time-series history |
| `ticketing_dataset` | Customer support intent classification |

### Example

Source: `app/models/request_models.py` -- `Example(BaseModel)`.

Few-shot example structure for SFT generation.

| Field | Type | Description |
|---|---|---|
| `question` | `str` | The question or prompt text |
| `solution` | `str` | The expected answer or completion |

### Example\_eval

Source: `app/models/request_models.py` -- `Example_eval(BaseModel)`.

Few-shot example structure for evaluation scoring.

| Field | Type | Description |
|---|---|---|
| `score` | `float` | Numeric quality score |
| `justification` | `str` | Text explanation for the assigned score |

### ModelParameters

Source: `app/models/request_models.py` -- `ModelParameters(BaseModel)`.

Low-level generation parameters passed to the LLM provider. When `model_params` is `None` on a request, provider-specific defaults apply.

| Field | Type | Default | Range | Description |
|---|---|---|---|---|
| `temperature` | `float` | `0.0` | 0.0--2.0 | Controls randomness of output |
| `top_p` | `float` | `1.0` | 0.0--1.0 | Nucleus sampling threshold |
| `min_p` | `float` | `0.0` | 0.0--1.0 | Minimum probability threshold |
| `top_k` | `int` | `150` | >= 0 | Top-K sampling parameter |
| `max_tokens` | `int` | `8192` | >= 1 | Maximum tokens to generate |

### Export\_synth

Source: `app/models/request_models.py` -- `Export_synth(BaseModel)`.

Request body for exporting generated data to external destinations.

| Field | Type | Default | Description |
|---|---|---|---|
| `export_type` | `List[str]` | `["huggingface"]` | Export destinations (e.g., `["huggingface", "s3"]`) |
| `file_path` | `str` | **required** | Path to the generated data file |
| `display_name` | `str \| None` | `None` | Human-readable label |
| `output_key` | `str \| None` | `"Prompt"` | Key field name in the data |
| `output_value` | `str \| None` | `"Completion"` | Value field name in the data |
| `hf_config` | `HFConfig \| None` | `None` | Hugging Face export configuration |
| `s3_config` | `S3Config \| None` | `None` | S3 export configuration |

### HFConfig

Source: `app/models/request_models.py` -- `HFConfig(BaseModel)`.

Configuration for exporting datasets to Hugging Face Hub.

| Field | Type | Default | Description |
|---|---|---|---|
| `hf_repo_name` | `str` | **required** | Repository name on Hugging Face |
| `hf_username` | `str` | **required** | Hugging Face username |
| `hf_token` | `str` | **required** | Hugging Face API token |
| `hf_commit_message` | `str \| None` | `"Hugging face export"` | Commit message for the push |

### S3Config

Source: `app/models/request_models.py` -- `S3Config(BaseModel)`.

Configuration for exporting datasets to Amazon S3.

| Field | Type | Default | Description |
|---|---|---|---|
| `bucket` | `str` | **required** | S3 bucket name |
| `key` | `str` | `""` | S3 object key (path within the bucket) |
| `create_if_not_exists` | `bool` | `True` | Create the bucket if it does not exist |

### CustomPromptRequest

Source: `app/models/request_models.py` -- `CustomPromptRequest(BaseModel)`.

Request body for `POST /create_custom_prompt`, which generates an AI-enhanced prompt from user instructions.

| Field | Type | Default | Description |
|---|---|---|---|
| `model_id` | `str` | **required** | LLM model identifier |
| `custom_prompt` | `str` | **required** | User's prompt instructions |
| `inference_type` | `str \| None` | `"aws_bedrock"` | LLM provider backend |
| `caii_endpoint` | `str \| None` | `None` | CAII endpoint URL |
| `example_path` | `str \| None` | `None` | Path to an external examples file |
| `example` | `List[Dict[str, Any]] \| None` | `None` | Inline flexible-schema examples |
| `custom_p` | `bool` | `True` | Custom prompt flag |

### JsonDataSize

Source: `app/models/request_models.py` -- `JsonDataSize(BaseModel)`.

Request body for querying the size of input data before generation.

| Field | Type | Default | Description |
|---|---|---|---|
| `input_path` | `List[str]` | **required** | Paths to input files |
| `input_key` | `str \| None` | `"Prompt"` | Column name to read from the files |

### RelativePath

Source: `app/models/request_models.py` -- `RelativePath(BaseModel)`.

Utility model for resolving file paths relative to the project root.

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | `str \| None` | `""` | Relative path string |
