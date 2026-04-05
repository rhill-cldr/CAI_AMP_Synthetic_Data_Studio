# Model Handler Factory

Source: `app/core/model_handlers.py`, `app/core/config.py`

## 1. Factory Pattern

`create_handler()` returns a configured `UnifiedModelHandler` for the requested provider.

```python
def create_handler(
    model_id: str,
    bedrock_client=None,
    model_params: Optional[ModelParameters] = None,
    inference_type: Optional[str] = "aws_bedrock",
    caii_endpoint: Optional[str] = None,
    custom_p: bool = False,
) -> UnifiedModelHandler
```

| Parameter | Type | Default | Purpose |
|---|---|---|---|
| `model_id` | `str` | required | Provider-specific model identifier (e.g. `anthropic.claude-3-5-sonnet-20240620-v1:0`) |
| `bedrock_client` | boto3 client | `None` (auto-created) | Pre-configured `bedrock-runtime` client |
| `model_params` | `ModelParameters` | default params | Temperature, top_p, top_k, max_tokens |
| `inference_type` | `str` | `"aws_bedrock"` | Provider selector -- see routing table below |
| `caii_endpoint` | `str` | `None` | Base URL for CAII or OpenAI-compatible endpoints |
| `custom_p` | `bool` | `False` | When `True`, return raw text instead of parsed JSON |

## 2. Provider Routing

`generate_response()` dispatches to the handler method matching `inference_type`.

```d2
direction: down

inference_type: inference_type {
  shape: diamond
  style.fill: "#fff3e0"
}

bedrock: _handle_bedrock_request() {
  shape: rectangle
  style.fill: "#e3f2fd"
  label: "_handle_bedrock_request()\nboto3 bedrock-runtime converse API"
}

caii: _handle_caii_request() {
  shape: rectangle
  style.fill: "#e8f4f8"
  label: "_handle_caii_request()\nOpenAI SDK + CAII endpoint"
}

openai: _handle_openai_request() {
  shape: rectangle
  style.fill: "#e8f4f8"
  label: "_handle_openai_request()\nOpenAI SDK"
}

openai_compat: _handle_openai_compatible_request() {
  shape: rectangle
  style.fill: "#e8f4f8"
  label: "_handle_openai_compatible_request()\nOpenAI SDK + custom base_url"
}

gemini: _handle_gemini_request() {
  shape: rectangle
  style.fill: "#f0e8f8"
  label: "_handle_gemini_request()\ngoogle-generativeai SDK"
}

error: ModelHandlerError {
  shape: rectangle
  style.fill: "#ffebee"
  label: "ModelHandlerError\n(unsupported inference_type)"
}

inference_type -> bedrock: "aws_bedrock"
inference_type -> caii: "CAII"
inference_type -> openai: "openai"
inference_type -> openai_compat: "openai_compatible"
inference_type -> gemini: "gemini"
inference_type -> error: "other" {
  style.stroke-dash: 3
}
```

| `inference_type` | Handler Method | SDK | Authentication |
|---|---|---|---|
| `aws_bedrock` | `_handle_bedrock_request` | boto3 `bedrock-runtime` | AWS credentials (env vars or instance profile) |
| `CAII` | `_handle_caii_request` | OpenAI (custom endpoint) | `CDP_TOKEN` env var or `/tmp/jwt` file |
| `openai` | `_handle_openai_request` | OpenAI | `OPENAI_API_KEY` env var |
| `openai_compatible` | `_handle_openai_compatible_request` | OpenAI (custom `base_url`) | `OpenAI_Endpoint_Compatible_Key` env var |
| `gemini` | `_handle_gemini_request` | google-generativeai | `GEMINI_API_KEY` env var |

All non-Bedrock providers use the OpenAI chat completions API shape (`messages=[{"role":"user","content":prompt}]`). Bedrock uses the `converse` API with its own message format.

When `custom_p=False` (default), every handler passes the response through `_extract_json_from_text()` before returning. When `custom_p=True`, raw text is returned as-is.

## 3. Model Family Detection

`get_model_family(model_id)` in `app/core/config.py` maps model identifiers to a `ModelFamily` enum used for prompt formatting.

| Pattern Matched in `model_id` | Returns | Prompt Format |
|---|---|---|
| `anthropic.claude` or `us.anthropic.claude` | `ModelFamily.CLAUDE` | Passthrough (no wrapping) |
| `meta.llama`, `us.meta.llama`, or `meta/llama` | `ModelFamily.LLAMA` | Llama special tokens |
| `mistral` or `mistralai/` | `ModelFamily.MISTRAL` | `[INST]...[/INST]` wrapping |
| `Qwen` or `qwen` | `ModelFamily.QWEN` | `<\|im_start\|>` / `<\|im_end\|>` tokens |
| (none of the above) | `model_id.split('/')[-1]` | Best-effort from model name |

```python
class ModelFamily(str, Enum):
    CLAUDE  = "claude"
    LLAMA   = "llama"
    MISTRAL = "mistral"
    QWEN    = "qwen"
```

## 4. Retry and Backoff

AWS-style exponential backoff applies to Bedrock requests only.

| Constant | Value |
|---|---|
| `MAX_RETRIES` | 2 |
| `BASE_DELAY` | 3 seconds |
| `MULTIPLIER` | 1.5 |

Delay sequence: **3 s --> 4.5 s --> 6.75 s** (computed as `BASE_DELAY * MULTIPLIER ^ retry_count`).

Retried error types:

| Error | Behavior |
|---|---|
| `ThrottlingException` | Retry with backoff |
| `ServiceUnavailableException` | Retry with backoff |
| `ConnectionClosedError` | Retry with backoff; recreate boto3 client |
| `EndpointConnectionError` | Retry with backoff; recreate boto3 client |
| `ProtocolError` | Retry with backoff; recreate boto3 client |
| `ValidationException` ("too many input tokens") | Retry with `max_tokens` halved (8192 --> 4096 --> 2048) |
| `ValidationException` ("model identifier is invalid") | Raise `InvalidModelError` immediately (no retry) |

## 5. Timeout Configuration

All providers use a 1-hour read timeout to accommodate long generation requests.

| Provider | Connect Timeout | Read Timeout | Write Timeout | Pool Timeout |
|---|---|---|---|---|
| AWS Bedrock | 5 s | 3600 s | (boto3 default) | (boto3 default) |
| OpenAI | 5 s | 3600 s | 10 s | 5 s |
| OpenAI Compatible | 5 s | 3600 s | 10 s | 5 s |
| CAII | 5 s | 3600 s | 10 s | 5 s |
| Gemini | -- | 3600 s | -- | -- |

OpenAI, OpenAI Compatible, and CAII handlers configure timeouts via `httpx.Timeout` and pass a custom `httpx.Client` to the OpenAI SDK. When `/etc/ssl/certs/ca-certificates.crt` exists, the httpx client uses it for TLS verification (private cloud support).

## 6. JSON Response Parsing

`_extract_json_from_text(text)` attempts to parse LLM output into `List[Dict[str, Any]]`. Steps are tried in order; the first success is returned.

| Step | Method | Handles |
|---|---|---|
| 1 | `json.loads(text)` | Clean JSON response |
| 2 | Find `[...]` boundaries, `json.loads()` on substring | JSON embedded in surrounding prose |
| 3 | `ast.literal_eval()` on extracted substring | Python-literal JSON variants (single quotes) |
| 4 | Text cleaning (strip newlines, replace `'` with `"`, remove tabs) then `json.loads()` | Whitespace / quote issues |
| 5 | Regex: `"score":\s*(\d+\.?\d*),\s*"justification":\s*"([^"]*)"` | Evaluation responses |
| 6 | Regex: `"question":\s*"([^"]*)",\s*"solution":\s*"([^"]*)"` | Generation Q&A pairs |

Returns empty list `[]` if all parsing fails, with the exception that step 2-4 failures fall through to step 5-6, and if regex matches are found those are returned. If no regex matches either, returns `[{"text": text}]` wrapping the raw response.

## 7. CAII Authentication

`_get_caii_token()` in `app/core/config.py` resolves the bearer token for Cloudera AI Inference endpoints.

Resolution order:

```
1. CDP_TOKEN environment variable  -->  use directly
2. /tmp/jwt file                   -->  parse JSON, extract "access_token"
3. Neither available               -->  HTTP 401 Unauthorized
```

Health check (`caii_check()`):

| Parameter | Value |
|---|---|
| Method | `GET {endpoint}/models` |
| Timeout | 3 seconds |
| Auth header | `Bearer {token}` from `_get_caii_token()` |
| Failure codes | 401/403 --> token access error; 404 --> bad endpoint; other --> raise `HTTPException(503)` |
