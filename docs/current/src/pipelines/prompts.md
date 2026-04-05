# Prompt Engineering System

Source: `app/core/prompt_templates.py`

## Overview

Three classes handle prompt construction for all generation and evaluation pipelines.

| Class | Role | Typical Caller |
|---|---|---|
| `PromptHandler` | Static utility methods -- format examples, resolve default schemas, prompts, and examples | `ModelPrompts` |
| `ModelPrompts` | Builds the actual prompt strings and wraps them for the target model family | `PromptBuilder` |
| `PromptBuilder` | Thin facade that delegates every call to `ModelPrompts` | Service layer (`SynthesisService`, `EvaluatorService`, etc.) |

Services never construct prompts directly. They call `PromptBuilder`, which forwards to `ModelPrompts`, which calls `PromptHandler` to resolve any missing defaults before assembling the final string.

## Prompt Assembly

```d2
direction: down

caller: Service Layer {
  label: "SynthesisService\nor EvaluatorService"
}

builder: PromptBuilder {
  label: "PromptBuilder\n(facade)"
}

handler: PromptHandler {
  label: "PromptHandler\nformat_examples()\nget_default_*()"
}

prompts: ModelPrompts {
  label: "ModelPrompts\nget_generate_prompt()\nget_eval_prompt()\nget_freeform_prompt()\ncreate_custom_prompt()"
}

family: Model Family {
  claude: Claude
  llama: Llama
  mistral: Mistral
  qwen: Qwen
  other: Other
}

caller -> builder
builder -> prompts
prompts -> handler: resolve defaults
prompts -> family: format for model
```

---

## PromptBuilder Methods

`PromptBuilder` is the only class services import. Each method delegates to a single `ModelPrompts` method without additional logic.

| Method | Delegates To | Used By |
|---|---|---|
| `build_prompt()` | `ModelPrompts.get_generate_prompt()` | SFT generation |
| `build_eval_prompt()` | `ModelPrompts.get_eval_prompt()` | Legacy evaluation |
| `build_generate_result_prompt()` | `ModelPrompts.generate_result_prompt()` | Custom\_Workflow generation |
| `build_custom_prompt()` | `ModelPrompts.create_custom_prompt()` | Prompt auto-generation |
| `build_freeform_prompt()` | `ModelPrompts.get_freeform_prompt()` | Freeform generation |
| `build_freeform_eval_prompt()` | `ModelPrompts.get_freeform_eval_prompt()` | Freeform evaluation |

---

## Model-Family Formatting

Every prompt method in `ModelPrompts` ends with the same dispatch block. `get_model_family(model_id)` (from `app/core/config.py`) returns a `ModelFamily` enum, and the prompt string is wrapped accordingly.

| Family | Format |
|---|---|
| Claude | Raw prompt (no wrapper tokens) |
| Llama | `<\|begin_of_text\|><\|start_header_id\|>user<\|end_header_id\|>` ... `<\|eot_id\|><\|start_header_id\|>assistant<\|end_header_id\|>` |
| Mistral | `[INST]` ... `[/INST]` |
| Qwen | `<\|im_start\|>system` ... `<\|im_end\|>` `<\|im_start\|>user` ... `<\|im_end\|>` `<\|im_start\|>assistant` |
| Other | Same as Claude (raw prompt, no wrapper tokens) |

The Qwen format also injects a system prompt (`"You are Qwen, created by Alibaba Cloud. You are a helpful assistant."`) before the user content.

---

## Generation Prompt Structure (SFT)

Built by `ModelPrompts.get_generate_prompt()`. Called via `PromptBuilder.build_prompt()`.

The assembled prompt contains five sections in order:

1. **Base prompt** -- `"You are a very helpful assistant which creates a valid json array based on instructions given"`
2. **Task-specific section** -- varies by `use_case`:

| Use Case | Section Content |
|---|---|
| `code_generation` | `"Create {n} programming question-solution pairs about {topic}"` |
| `text2sql` | `"Using this database schema: {schema}\nCreate {n} natural language to SQL query pairs about {topic}"` |
| `custom` | `"Create {n} question-solution pairs about {topic}"` |

3. **Custom prompt** -- user-provided string, or a per-use-case default from module constants (`DEFAULT_CODE_GENERATION_PROMPT`, `DEFAULT_TEXT2SQL_PROMPT`, or `" "` for custom).
4. **Omit prompt** -- pipe-separated list of already-generated questions to prevent duplicates (see [Deduplication](#deduplication) below).
5. **JSON instruction** -- strict formatting requirements (double quotes only, proper escaping, no trailing commas, no surrounding text) plus formatted examples in `<examples>` tags.

The custom prompt and JSON instruction are concatenated into a single `json_prompt` string that follows the base prompt.

---

## Custom\_Workflow Prompt Structure

Built by `ModelPrompts.generate_result_prompt()`. Called via `PromptBuilder.build_generate_result_prompt()`.

This prompt is used when a seed input (question) already exists and the LLM must produce a completion (solution).

1. **Base prompt** -- `"You are a very helpful assistant. Observe in the given examples how a solution is provided for a given question."` followed by examples in `<examples>` tags.
2. **Instruction bridge** -- `"Now that you have looked at examples you have to follow instructions very carefully and generate response based on those instructions."`
3. **Custom prompt** -- user-provided or per-use-case default (same resolution as SFT).
4. **Task-specific section** -- varies by `use_case`:

| Use Case | Section Content |
|---|---|
| `code_generation` | `"Give a programming solution for following: <input>{input}</input>"` |
| `text2sql` | `"Using this database schema: {schema}\nCreate a SQL query for: <input>{input}</input>"` |
| `custom` | `"Create a solution about the following based on above instructions: <input>{input}</input>"` |

Unlike SFT, there is no JSON instruction block and no omit list. The model handler is created with `custom_p=True`, so responses are returned as raw text.

---

## Freeform Prompt Structure

Built by `ModelPrompts.get_freeform_prompt()`. Called via `PromptBuilder.build_freeform_prompt()`.

1. **Base prompt** -- same assistant preamble as SFT, plus `"Create {n} set of data about {topic} based on the instructions provided below"`.
2. **Instructions** -- user-provided custom prompt or the default from `USE_CASE_CONFIGS[use_case].prompt`, wrapped in `<instructions>` tags.
3. **Text2SQL special case** -- when `use_case == TEXT2SQL`, the database schema is prepended before the `<instructions>` block.
4. **Omit prompt** -- same deduplication mechanism as SFT (see [Deduplication](#deduplication)).
5. **JSON instruction** -- same strict formatting requirements, with examples in `<examples>` tags.

### Example Resolution (Freeform)

Examples are resolved in priority order:

| Priority | Source | Condition |
|---|---|---|
| 1 | `example_path` (file) | File path provided; loaded via `DataLoader.load()` (limited to 10 rows) |
| 2 | `example_custom` (JSON) | List of dicts provided directly on the request |
| 3 | `USE_CASE_CONFIGS` defaults | Fallback when neither file nor custom examples are provided |

File-based examples are loaded as a DataFrame, converted to a list of dicts, and serialized with custom handling for pandas/numpy types (`Timestamp`, `datetime64`, `integer`, `floating`, `ndarray`).

---

## Evaluation Prompt Structure

### Legacy Evaluation

Built by `ModelPrompts.get_eval_prompt()`. Called via `PromptBuilder.build_eval_prompt()`.

1. **Judge persona** -- `"You are a brilliant judge on evaluating quality of question and answer pair. Follow the given instructions below to evaluate given question and answer pair."`
2. **Custom evaluation rubric** -- from `USE_CASE_CONFIGS_EVALS[use_case].prompt` or user-provided. Default rubrics use an additive 5-point scoring system with per-point criteria.
3. **Item to evaluate** -- `Question: {question}` and `Solution: {solution}`.
4. **JSON output format** -- must return `[{"score": N, "justification": "..."}]` with double quotes, proper escaping, and no surrounding text.
5. **Few-shot examples** -- formatted evaluation examples showing expected score/justification pairs.

### Freeform Evaluation

Built by `ModelPrompts.get_freeform_eval_prompt()`. Called via `PromptBuilder.build_freeform_eval_prompt()`.

Follows the same structure as legacy evaluation with two differences:

| Aspect | Legacy | Freeform |
|---|---|---|
| Judge persona | "...quality of question and answer pair" | "...evaluating a set of data with fields and corresponding values" |
| Item format | Separate `Question:` and `Solution:` fields | Entire row dict passed as `data row: {row}` |

Example resolution for evaluation uses `USE_CASE_CONFIGS_EVALS[use_case].default_examples` as the fallback when no examples are provided.

---

## Custom Prompt Generation

`ModelPrompts.create_custom_prompt()` auto-generates prompts by analyzing example data. Called via `PromptBuilder.build_custom_prompt()`.

### Pipeline

1. **Load example data** (if `example_path` or `example` is provided):
   - File path: loaded via `DataLoader.load()`.
   - JSON list: converted to a DataFrame via `pd.DataFrame(example)`.
   - Data types are inferred via `DataLoader.infer_dtypes()`.

2. **Analyze data** via `DataAnalyser.analyse()`:
   - Produces a summary dict containing `columns`, `statistical_analysis` (numeric, categorical, datetime), `cross_row_relationship`, and `cross_column_relationship`.
   - The summary is wrapped in `<data_summary>` tags with instructions to match distributions and relationships.

3. **Create CSV snippet** via `SummaryFormatter.first_rows_block()`:
   - First 10 rows of the original dataset.
   - Wrapped in `<example>` tags with instructions to preserve column order, header names, and data types.

4. **Construct the final instruction**:
   - When `example_path` is provided: includes column list requirements, statistical analysis inclusion, and references to Code Generation and Lending Data prompts as formatting examples.
   - When only JSON examples are provided: a simpler instruction referencing Code Generation and Text2SQL prompts as formatting examples.
   - Both variants end with: `"Make sure you just give the prompt in your response which can be directly used by large language model."`

5. **Wrap for model family** using the same dispatch block as all other prompt methods.

---

## PromptHandler Defaults Resolution

`PromptHandler` provides static methods that resolve missing request parameters to sensible defaults. Each method follows a consistent pattern: use the request value if present, otherwise fall back to a configured default.

| Method | Resolution Order |
|---|---|
| `get_default_example()` | Request examples --> `USE_CASE_CONFIGS[use_case].default_examples` --> hardcoded custom fallback (capital-of-France Q&A pair) |
| `get_default_schema()` | Request schema --> `DEFAULT_SCHEMA` (text2sql only) --> `None` (code\_generation, custom) |
| `get_default_custom_prompt()` | Request prompt --> `DEFAULT_TEXT2SQL_PROMPT` / `DEFAULT_CODE_GENERATION_PROMPT` --> `" "` (custom) |
| `get_default_eval_example()` | Request examples --> hardcoded per-use-case evaluation examples (score/justification pairs) |
| `get_default_custom_eval_prompt()` | Request prompt --> `USE_CASE_CONFIGS_EVALS[use_case].prompt` |
| `get_freeform_default_custom_prompt()` | Request prompt --> `DEFAULT_freeform_TEXT2SQL_PROMPT` / `DEFAULT_freeform_CODE_GENERATION_PROMPT` --> `" "` (custom) |

### Default Schema

The `DEFAULT_SCHEMA` used for text2sql when no schema is provided defines six tables:

| Table | Key Columns |
|---|---|
| `employees` | `id`, `name`, `department`, `salary` |
| `departments` | `id`, `name`, `manager_id`, `budget` |
| `customers` | `customer_id`, `first_name`, `last_name`, `email`, `phone`, `address`, `city`, `country`, `registration_date`, `total_orders` |
| `products` | `product_id`, `name`, `description`, `category`, `price`, `stock_quantity`, `supplier_id`, `created_at`, `is_active` |
| `orders` | `order_id`, `customer_id`, `order_date`, `status`, `shipping_address`, `shipping_cost`, `total_amount`, `payment_method` |
| `order_items` | `order_id`, `product_id`, `quantity`, `unit_price`, `discount` |

---

## Deduplication

The omit list prevents the LLM from regenerating previously produced items within the same generation run.

### How It Works

1. After each successful batch, generated items are added to a per-topic omit list.
2. The omit list is included in the next prompt as a pipe-separated string.
3. The list is trimmed to the **last 100 entries** per topic to avoid exceeding context limits.

### Key Extraction

| Technique | Extraction Logic |
|---|---|
| SFT | Collects `pair["question"]` from each generated QA pair |
| Freeform | Collects the first string field found by checking keys in order: `id`, `name`, `title`, `question`, `prompt`, `key`. Falls back to the first string value in the dict if none match. |

### Prompt Wording

| Technique | Omit Instruction |
|---|---|
| SFT | `"Make it absolutely sure that you don't include questions mentioned in below list as we already have question pair solutions for them."` |
| Freeform | `"Following is the list of corresponding values for given fields you have already created. For each item you generate, verify it's distinct from others in the current list by comparing key field. Create NEW items that are substantially different."` |

When the omit list is empty, a single space is used as the omit prompt (effectively no-op).

Source: `app/core/prompt_templates.py`
