# System Overview

Synthetic Data Studio is composed of four layers that communicate over well-defined boundaries. A **React frontend** (Vite build, served as static files) sends requests to a **FastAPI application** (`main.py` -- routes, middleware, CORS). The application delegates to a **Service layer** containing synthesis, evaluation, alignment, and export pipelines. Services depend on a **Core layer** that provides a provider-agnostic model handler, prompt templates, a SQLite database manager, path resolution, and configuration.

## Component Topology

```d2
frontend: React Frontend {
  label: "React Frontend\nVite build, static files served by FastAPI"
}

app: FastAPI Application {
  label: "FastAPI Application\nmain.py -- routes, middleware, CORS"
}

services: Service Layer {
  synth: SynthesisService {
    label: "SynthesisService\n(Freeform)"
  }
  synth_legacy: SynthesisLegacyService {
    label: "SynthesisLegacyService\n(SFT, Custom_Workflow)"
  }
  eval: EvaluatorService {
    label: "EvaluatorService\n(Freeform)"
  }
  eval_legacy: EvaluatorLegacyService {
    label: "EvaluatorLegacyService\n(SFT, Custom_Workflow)"
  }
  alignment: ModelAlignment
  export: Export_Service
}

core: Core Layer {
  handler: UnifiedModelHandler {
    label: "UnifiedModelHandler\nmodel_handlers.py"
  }
  prompts: PromptBuilder {
    label: "PromptBuilder\nprompt_templates.py"
  }
  db: DatabaseManager {
    label: "DatabaseManager\ndatabase.py"
  }
  paths: PathManager {
    label: "PathManager\npath_manager.py"
  }
  config: Config {
    label: "Config\nconfig.py"
  }
}

external: External Systems {
  bedrock: AWS Bedrock
  openai: OpenAI API
  gemini: Google Gemini
  caii: Cloudera AI Inference
  sqlite: SQLite (metadata.db) {
    shape: cylinder
  }
  hf: HuggingFace Hub
  s3: Amazon S3
  cml: CML Jobs API
}

frontend -> app: HTTP / REST
app -> services.synth
app -> services.synth_legacy
app -> services.eval
app -> services.eval_legacy
app -> services.alignment
app -> services.export

services.synth -> core.handler
services.synth -> core.prompts
services.synth_legacy -> core.handler
services.synth_legacy -> core.prompts
services.eval -> core.handler
services.eval_legacy -> core.handler
services.alignment -> core.handler
services.export -> core.db

core.handler -> external.bedrock
core.handler -> external.openai
core.handler -> external.gemini
core.handler -> external.caii
core.db -> external.sqlite
services.export -> external.hf
services.export -> external.s3
app -> external.cml: batch job dispatch
```

## Service Routing

The technique field on each request determines which service handles it. Evaluation follows the same legacy/freeform split.

| Technique | Service | Entry Point | Purpose |
|---|---|---|---|
| `SFT` | `SynthesisLegacyService` | `generate_examples()` | Topic to QA pairs |
| `Custom_Workflow` | `SynthesisLegacyService` | `generate_result()` | Seed input to completions |
| `Freeform` | `SynthesisService` | `generate_freeform()` | Open-ended structured output |
| `Model_Alignment` | `ModelAlignment` | composite pipeline | Generate alternates, evaluate, produce DPO/KTO |
| `SFT` / `Custom_Workflow` (eval) | `EvaluatorLegacyService` | `evaluate()` | Score legacy outputs |
| `Freeform` (eval) | `EvaluatorService` | `evaluate()` | Score freeform outputs |

## Deployment Contexts

| | Local Development | Cloudera ML |
|---|---|---|
| **Detection** | Default | `CDSW_PROJECT_ID` env var present |
| **Database** | SQLite in project root | SQLite in project root |
| **Port** | 8100 | 8100 |
| **Batch mode** | Disabled (preview only) | CML Jobs API for async execution |
| **Path prefix** | None | `synthetic-data-studio/` when `IS_COMPOSABLE=true` (via `PathManager`) |
| **Job management** | N/A | CML API creates, monitors, and cancels jobs |

## Design Principles

1. **Single-file routing.** All API routes live in `main.py`. No router modules, no blueprint registration.
2. **Service layer split.** Legacy (structured) and Freeform (open-ended) paths run identical pipelines with different prompt construction.
3. **Provider-agnostic model handling.** `UnifiedModelHandler` plus a factory pattern abstracts all LLM providers behind a single `generate_response()` interface.
4. **Dual execution modes.** Preview (synchronous, <=25 items) and Batch (async CML Jobs) share the same service code.
5. **Metadata tracking.** Every generation, evaluation, and export is recorded in SQLite for history and auditability.
