# Configuration & Discovery Routes

These endpoints return provider catalogs, model parameters, use-case metadata,
prompt templates, and file utilities.  None of them mutate server state.

---

## Model Discovery

### GET /model/model_ID

List available model IDs grouped by provider.  No health check is performed;
the response is fast but may include models that are temporarily unreachable.

**Response**

```json
{
  "models": {
    "aws_bedrock": ["us.anthropic.claude-3-5-haiku-...", "..."],
    "CAII": [],
    "OpenAI": [],
    "Google Gemini": []
  }
}
```

Bedrock models are populated by `list_bedrock_models()`, which queries the
Bedrock `ListFoundationModels` API filtered to:

| Filter | Value |
|---|---|
| Inference type | `ON_DEMAND` |
| Input modality | `TEXT` |
| Output modality | `TEXT` |
| Provider | Anthropic, Meta, Mistral AI |

Inference profiles matching those providers are appended.  The combined list is
deduplicated and sorted newest-first by `sort_unique_models()`.

---

### GET /model/model_id_filter

List models with a health-check status (`enabled` / `disabled`) per provider.

**Response**

```json
{
  "models": {
    "aws_bedrock": {"enabled": [...], "disabled": [...]},
    "openai":      {"enabled": [...], "disabled": [...]},
    "google_gemini": {"enabled": [...], "disabled": [...]},
    "CAII":        {"enabled": [...], "disabled": [...]}
  }
}
```

All providers are probed concurrently via `collect_model_catalog()`.

| Provider | Probe method | Timeout | Concurrency | Requirements |
|---|---|---|---|---|
| AWS Bedrock | `generate_response()` with minimal params | 5 s | 10 | AWS credentials configured |
| OpenAI | `chat.completions.create()` | 5 s | 5 | `OPENAI_API_KEY` env var |
| Google Gemini | `generate_content()` | 5 s | 5 | `GEMINI_API_KEY` env var |
| CAII | `caii_check()` on each endpoint | 3 s | 5 | On-cluster only (`CDSW_PROJECT_ID` set) |

If a required env var is missing the provider returns all models as `disabled`.

**OpenAI curated model list:**
`gpt-4.1`, `gpt-4.1-mini`, `gpt-4.1-nano`, `o3`, `o4-mini`, `o3-mini`, `o1`, `gpt-4o`, `gpt-4o-mini`, `gpt-4-turbo`, `gpt-3.5-turbo`

**Gemini curated model list:**
`gemini-2.5-pro`, `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-2.0-flash`, `gemini-2.0-flash-lite`, `gemini-1.5-pro`, `gemini-1.5-flash`, `gemini-1.5-flash-8b`

**CAII** is only queried when running inside a CML project.  It lists loaded
`TEXT_GENERATION` endpoints via `cmlapi` and returns objects with `model` and
`endpoint` fields rather than plain strings.

---

### GET /model/parameters

Return model parameter ranges and defaults.

**Response**

```json
{
  "parameters": {
    "temperature": {"min": 0.0, "max": 2.0, "default": 0.0},
    "top_p":       {"min": 0.0, "max": 1.0, "default": 1.0},
    "top_k":       {"min": 0,   "max": 300, "default": 50},
    "max_tokens":  {"min": 1,   "max": 8192, "default": 8192}
  }
}
```

---

## Use-Case Metadata

### GET /use-cases

List all available use cases with descriptions and tags.

**Response**

```json
{
  "usecases": [
    {
      "id": "code_generation",
      "name": "Code Generation",
      "description": "...",
      "tag": ["Supervised Finetuning", "Data Generation"]
    }
  ]
}
```

| Use Case ID | Name | Tags |
|---|---|---|
| `code_generation` | Code Generation | Supervised Finetuning, Data Generation |
| `text2sql` | Text to SQL | Supervised Finetuning, Data Generation |
| `custom` | Custom | (none) |
| `lending_data` | Lending Data | Data Generation, Tabular Data |
| `credit_card_data` | Credit Card Data | Data Generation, Tabular Data |
| `ticketing_dataset` | Ticketing Dataset | Data Generation, Intent Classification |

---

### GET /use-cases/{use_case}/topics

Get available topics for a specific use case.

**Response:** `{"topics": ["Python Basics", "Data Manipulation", ...]}`

Topics are loaded from `USE_CASE_CONFIGS[use_case].topics`.

---

## Prompt & Example Templates

| Route | Method | Returns |
|---|---|---|
| `/{use_case}/gen_prompt` | GET | Default generation prompt for the use case |
| `/{use_case}/gen_freeform_prompt` | GET | Default freeform generation prompt |
| `/{use_case}/eval_prompt` | GET | Default evaluation prompt |
| `/{use_case}/gen_examples` | GET | `{"examples": [...]}` -- default generation examples (empty list for `custom`) |
| `/{use_case}/eval_examples` | GET | `{"examples": [...]}` -- default evaluation examples (empty list for `custom`) |
| `/{use_case}/custom_gen_examples` | GET | Code snippets (`code_generation`), SQL queries (`text2sql`), or empty list (`custom`) |
| `/{use_case}/example_payloads` | GET | A complete `SynthesisRequest` example for the use case |

---

### GET /sql_schema

Get the default SQL schema for the `text2sql` use case.  Delegates to
`PromptHandler.get_default_schema("text2sql")`.

---

## Health Check

### GET /health

Health check endpoint.  Delegates to
`SynthesisLegacyService.get_health_check()`.  The global HTTP middleware
enforces a 5-second timeout on this route.

---

## Data File Routes

### POST /json/dataset_size

Get the total number of items in a dataset.

**Request body** (`JsonDataSize`):

| Field | Type | Default | Description |
|---|---|---|---|
| `input_path` | `List[str]` | required | List of JSON file paths |
| `input_key` | `string` | `"Prompt"` | Key to extract from each item |

**Response:** `{"dataset_size": N}`

Returns an error (400) if any file contains invalid JSON, is empty, or is
missing the specified key.

---

### POST /json/get_content

Get a preview of file content (max 20 rows).  Supports `.json` and `.csv`
files.  Nested JSON objects are flattened with dot notation.

**Request body** (`RelativePath`):

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | `string` | `""` | Path to the file |

**Response:** `{"data": [...]}`

---

### POST /json/get_seeds_list

Get raw JSON content from a file without truncation or flattening.

**Request body** (`RelativePath`):

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | `string` | `""` | Path to the JSON file |

**Response:** `{"data": [...]}`

---

### POST /get_project_files

List project files.  Only available on CML (`CDSW_PROJECT_ID` set).
Delegates to `client_cml.list_project_files()`.

**Request body** (`RelativePath`):

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | `string` | `""` | Relative path within the project |
