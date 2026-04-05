# Use Case Configuration

Source: `app/core/config.py`

The use case configuration system defines the six built-in data generation
scenarios, their default topics and few-shot examples, the evaluation rubrics
used to score generated data, model family detection logic, and a curated
Bedrock model catalog.  All configuration is loaded at import time from the
`USE_CASE_CONFIGS` and `USE_CASE_CONFIGS_EVALS` dictionaries.

---

## UseCase Enum

`UseCase(str, Enum)` identifies each built-in scenario.  The string value is
used as the API path parameter and database key.

| Value | Label | Description |
|---|---|---|
| `code_generation` | Code Generation | Programming Q&A pairs with runnable code examples |
| `text2sql` | Text to SQL | Natural-language questions matched to SQL queries |
| `custom` | Custom | Blank template for user-defined tasks |
| `lending_data` | Lending Data | LendingClub-style loan records with cross-field consistency |
| `credit_card_data` | Credit Card Data | User profiles with chronological credit-status histories |
| `ticketing_dataset` | Ticketing Dataset | Customer-support prompts with intent classification |

---

## USE_CASE_CONFIGS

Each use case maps to a `UseCaseMetadata` instance bundling the following
fields:

| Field | Type | Purpose |
|---|---|---|
| `name` | `str` | Human-readable display name |
| `description` | `str` | Multi-sentence description shown in the UI |
| `topics` | `list[str]` | Default topic list (empty for `custom`) |
| `default_examples` | `List[Dict]` | Few-shot examples sent to the LLM |
| `prompt` | `Optional[str]` | Default generation prompt (`None` or string) |
| `schema` | `Optional[str]` | SQL schema (only populated for `text2sql`) |

### Default Topics by Use Case

| Use Case | Default Topics |
|---|---|
| `code_generation` | Python Basics, Data Manipulation, Web Development, Machine Learning, Algorithms |
| `text2sql` | Basic Queries, Joins, Aggregations, Subqueries, Windows Functions |
| `custom` | *(none)* |
| `lending_data` | Business loans, Personal loans, Auto loans, Home equity loans, Asset-backed loans |
| `credit_card_data` | High income person, Low income person, Four-person family, Three-person family, Two-person family, Five-person family, more than 10 credit records, more than 20 credit records |
| `ticketing_dataset` | Technical Issues, Billing Queries, Payment queries |

### Default Examples

Each use case ships with few-shot examples that establish the expected output
format for the LLM.

| Use Case | Example Schema | Count |
|---|---|---|
| `code_generation` | `{"question": str, "solution": str}` | 2 |
| `text2sql` | `{"question": str, "solution": str}` | 2 |
| `custom` | *(empty list)* | 0 |
| `lending_data` | 26-field loan record (loan_amnt through address) | 4 |
| `credit_card_data` | 20-field flat row (ID through STATUS) | 5 |
| `ticketing_dataset` | `{"Prompt": str, "Completion": str}` | 3 |

### SQL Schema (text2sql only)

The `text2sql` use case includes a `DEFAULT_SQL_SCHEMA` containing two tables:

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL(10,2)
);

CREATE TABLE departments (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    manager_id INT,
    budget DECIMAL(15,2)
);
```

This schema is used by the default generation prompt and is returned by the
`GET /sql_schema` endpoint.

---

## USE_CASE_CONFIGS_EVALS

Each use case also maps to a `UseCaseMetadataEval` instance containing
evaluation rubrics.  The evaluation prompt instructs the LLM how to score
generated data, and the default examples provide few-shot calibration.

| Field | Type | Purpose |
|---|---|---|
| `name` | `str` | Display name (matches generation config) |
| `default_examples` | `List[Dict]` | Few-shot scoring examples with `score` and `justification` |
| `prompt` | `Optional[str]` | Full evaluation rubric sent to the LLM |

### Scoring Systems

| Use Case | Score Range | Scoring System |
|---|---|---|
| `code_generation` | 1--5 | Additive points: basic functionality (+1), correct implementation (+1), professional use (+1), best practices (+1), mastery (+1) |
| `text2sql` | 1--5 | Additive points: basic retrieval (+1), correct logic (+1), professional use (+1), efficiency (+1), mastery (+1) |
| `custom` | 1--5 | Generic quality assessment |
| `lending_data` | 1--10 | Deductive from 10: privacy (-1), formatting (-1), cross-column (-4), realism (-1), other (-2) |
| `credit_card_data` | 1--10 | Deductive from 10: privacy (-2), formatting (-1), data series (-4), cross-column (-1), other (-2) |
| `ticketing_dataset` | 1--5 | Keyword matching + quality assessment |

### code_generation Rubric (additive, 1--5)

| Point | Criterion |
|---|---|
| 1 | Code implements basic functionality and solves the core problem |
| 2 | Implementation is generally correct; may lack style refinement |
| 3 | Appropriate for professional use; demonstrates good Python patterns |
| 4 | Highly efficient; follows best practices with error handling and type hints |
| 5 | Outstanding mastery with comprehensive testing, security, and documentation |

### text2sql Rubric (additive, 1--5)

| Point | Criterion |
|---|---|
| 1 | Query retrieves the basic required data |
| 2 | Generally correct but may lack style consistency |
| 3 | Appropriate for professional use; good SQL pattern understanding |
| 4 | Highly efficient with proper indexing, joins, and formatting |
| 5 | Outstanding mastery with NULL handling, execution plan optimization, and scalability |

### lending_data Rubric (deductive, start at 10)

| Violation | Penalty |
|---|---|
| Privacy compliance (PII leakage) | -1 |
| Formatting inconsistencies (decimal precision, date format, term spacing) | -1 |
| Cross-column inconsistencies (grade/subgrade mismatch, mortgage vs. `mort_acc`, `open_acc` > `total_acc`) | -4 |
| Background knowledge / realism (unrealistic interest rates, implausible values) | -1 |
| Other violations | -2 |

Minimum score is capped at 1.  Critical errors (e.g., PII leakage, missing fields) also cap the score at 1.

### credit_card_data Rubric (deductive, start at 10)

| Violation | Penalty |
|---|---|
| Privacy (PII leakage) | -2 |
| Formatting (`CNT_CHILDREN`, `MONTHS_BALANCE` not integers, etc.) | -1 |
| Data series (chronological ordering, duplicate `MONTHS_BALANCE`, illogical status progression) | -4 |
| Cross-column (`FLAG_OWN_REALTY` vs. housing type, unemployed income, family member count) | -1 |
| Other violations | -2 |

Minimum score is capped at 1.

### ticketing_dataset Rubric (1--5)

Evaluation checks three dimensions:

1. **Query quality** -- professional language, respectful tone, customer service best practices.
2. **Keyword validity** -- response must be one of `cancel_ticket`, `customer_service`, or `report_payment_issue`.
3. **Match accuracy** -- the keyword must correctly classify the query intent.

If the response does not match any of the three valid keywords, the score is always 1.

---

## ModelFamily Enum

`ModelFamily(str, Enum)` classifies model identifiers into prompt-format
families.  Detection is performed by `get_model_family()`.

```python
class ModelFamily(str, Enum):
    CLAUDE  = "claude"
    LLAMA   = "llama"
    MISTRAL = "mistral"
    QWEN    = "qwen"
```

### Detection Logic

`get_model_family(model_id)` checks patterns against the `model_id` string in
order.  The first match wins.

| Family | Detection Pattern |
|---|---|
| `claude` | `"anthropic.claude"` or `"us.anthropic.claude"` in model_id |
| `llama` | `"meta.llama"`, `"us.meta.llama"`, or `"meta/llama"` in model_id |
| `mistral` | `"mistral"` or `"mistralai/"` in model_id |
| `qwen` | `"Qwen"` or `"qwen"` in model_id |
| *(fallback)* | Model name extracted from model_id (`model_id.split('/')[-1]`) |

The fallback returns a plain string rather than a `ModelFamily` enum member.

---

## Bedrock Model Catalog

The `bedrock_list` variable contains a curated list of Bedrock model
identifiers used for quick reference and health-check probing.

| Provider | Model | Identifier |
|---|---|---|
| Anthropic | Claude 3.5 Haiku | `us.anthropic.claude-3-5-haiku-20241022-v1:0` |
| Anthropic | Claude 3.5 Sonnet v2 | `us.anthropic.claude-3-5-sonnet-20241022-v2:0` |
| Anthropic | Claude 3 Opus | `us.anthropic.claude-3-opus-20240229-v1:0` |
| Anthropic | Claude Instant v1 | `anthropic.claude-instant-v1` |
| Meta | Llama 3.2 11B Instruct | `us.meta.llama3-2-11b-instruct-v1:0` |
| Meta | Llama 3.2 90B Instruct | `us.meta.llama3-2-90b-instruct-v1:0` |
| Meta | Llama 3.1 70B Instruct | `us.meta.llama3-1-70b-instruct-v1:0` |
| Mistral AI | Mixtral 8x7B Instruct | `mistral.mixtral-8x7b-instruct-v0:1` |
| Mistral AI | Mistral Large 2402 | `mistral.mistral-large-2402-v1:0` |
| Mistral AI | Mistral Small 2402 | `mistral.mistral-small-2402-v1:0` |

---

## Helper Functions

### get_available_topics(use_case)

```python
def get_available_topics(use_case: UseCase) -> Dict
```

Returns a dictionary containing the use case name, description, topic metadata,
and default examples.  Returns an empty dictionary if the use case is not found
in `USE_CASE_CONFIGS`.

**Return shape:**

```json
{
  "use_case": "Code Generation",
  "description": "...",
  "topics": { "<topic_id>": {"name": "...", "description": "...", "example_questions": [...]} },
  "default_examples": [...]
}
```

### get_examples_for_topic(use_case, topic)

```python
def get_examples_for_topic(use_case: UseCase, topic: str) -> List[Dict[str, str]]
```

Returns the example questions for a specific topic within a use case.  If the
topic is not found, falls back to the use case's `default_examples`.  Returns
an empty list if the use case itself is not found.

### caii_check(endpoint, timeout=3)

```python
def caii_check(endpoint: str, timeout: int = 3) -> requests.Response
```

Health-checks a Cloudera AI Inference endpoint by sending an authenticated
`GET` request to `{endpoint}/models`.

| Condition | Result |
|---|---|
| Endpoint is empty | `HTTPException(400)` |
| Request times out or connection fails | `HTTPException(503)` |
| Response status 401 or 403 | `HTTPException(403)` -- token lacks access |
| Response status 404 | `HTTPException(404)` -- endpoint not found |
| Response status 5xx | `HTTPException(503)` -- endpoint is downscaled |
| Response status 200 | Returns the `requests.Response` object |

Authentication is resolved by `_get_caii_token()`, which checks `CDP_TOKEN`
env var first, then falls back to parsing `/tmp/jwt`.
