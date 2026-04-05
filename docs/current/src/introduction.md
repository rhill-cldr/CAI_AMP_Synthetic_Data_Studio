# Introduction

Synthetic Data Studio (SDS) is a full-stack platform for generating, evaluating, and exporting synthetic datasets used to fine-tune large language models. It provides a FastAPI backend, a React frontend, and support for multiple LLM providers -- AWS Bedrock, OpenAI, Google Gemini, OpenAI-compatible endpoints, and Cloudera AI Inference (CAII). Datasets can be produced using several techniques: topic-based supervised fine-tuning (SFT), seed-input completion (Custom_Workflow), open-ended structured generation (Freeform), and preference-pair alignment (Model_Alignment for DPO/KTO). Results are scored by an LLM-as-judge evaluator and exported to HuggingFace Hub, Amazon S3, or the local project filesystem.

This guide serves two audiences:

| If you are... | Start here |
|---|---|
| **Building a synthesis client** to generate, evaluate, and export synthetic datasets via the SDS API | [Architecture Reference](./architecture/overview.md) and [API Specification](./api/conventions.md) |
| **Building a validation SDK or CI/CD pipeline** to validate synthetic data artifacts and automate generation workflows | [Data Models](./models/request-response.md) and [Integration & SDK](./integration/api-client.md) |

## Terminology

| Term | Definition |
|---|---|
| **Use Case** | A domain-specific configuration (`code_generation`, `text2sql`, `custom`, `lending_data`, `credit_card_data`, `ticketing_dataset`) that bundles default prompts, examples, topics, and evaluation rubrics. |
| **Technique** | The generation method -- `SFT` (topic-based Q&A), `Custom_Workflow` (seed-input completion), `Freeform` (open-ended structured data), or `Model_Alignment` (DPO/KTO preference pairs). |
| **Inference Type** | The LLM provider backend -- `aws_bedrock`, `CAII` (Cloudera AI Inference), `openai`, `openai_compatible`, or `gemini`. |
| **Preview Mode** | Synchronous inline execution (`is_demo=true`, up to 25 items). The API processes the request and returns results in the response body. |
| **Batch Mode** | Asynchronous execution via Cloudera ML Jobs (`is_demo=false`). The API returns a job ID; results are retrieved via history endpoints. |
| **Model Handler** | A `UnifiedModelHandler` instance created by the factory function `create_handler()`. Routes prompts to the appropriate LLM provider based on `inference_type`. |
| **Evaluation** | LLM-as-judge scoring of generated data. Each item receives a numeric score and text justification based on a configurable rubric. |
| **Model Alignment** | A composite pipeline that generates alternate completions, scores both versions, and produces DPO (chosen/rejected pairs) and KTO (binary preference) datasets. |
| **Export** | Pushing generated datasets to HuggingFace Hub, Amazon S3, or the local project filesystem. |

## Data Lifecycle

The following diagram shows the end-to-end flow from configuration through export.

```d2
direction: right

configure: Configure {
  shape: rectangle
  style.fill: "#e8f4f8"
  label: "Configure\n(use case + model + provider)"
}

generate: Generate {
  shape: rectangle
  style.fill: "#e8f4f8"
  label: "Generate\n(SFT / Custom_Workflow / Freeform)"
}

evaluate: Evaluate {
  shape: rectangle
  style.fill: "#e8f4f8"
  label: "Evaluate\n(LLM-as-judge scoring)"
}

align: Align (optional) {
  shape: rectangle
  style.fill: "#f0e8f8"
  label: "Align\n(DPO / KTO)"
}

export: Export {
  shape: rectangle
  style.fill: "#e8f8e8"
  label: "Export\n(HuggingFace / S3 / local)"
}

configure -> generate
generate -> evaluate
evaluate -> align
align -> export
evaluate -> export: skip alignment {
  style.stroke-dash: 3
}

# Execution mode branch
generate -> preview: "is_demo=true" {
  style.stroke: "#2196f3"
}
generate -> batch: "is_demo=false" {
  style.stroke: "#ff9800"
}

preview: Preview Mode {
  shape: rectangle
  style.fill: "#e3f2fd"
  label: "Preview Mode\n(inline, ≤25 items)"
}

batch: Batch Mode {
  shape: rectangle
  style.fill: "#fff3e0"
  label: "Batch Mode\n(CML Job, async)"
}

preview -> evaluate {
  style.stroke: "#2196f3"
}
batch -> evaluate {
  style.stroke: "#ff9800"
}
```

The API is the single integration surface -- everything documented in this guide can be driven by HTTP requests to the FastAPI backend.
