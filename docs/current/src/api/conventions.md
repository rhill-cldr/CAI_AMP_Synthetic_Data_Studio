# API Conventions

The SDS backend is a single FastAPI application titled "LLM Data Synthesis API" (version `1.0.0`). All routes are defined in `app/main.py` -- there are no router modules or blueprint registrations. The default port is **8100** (configurable via the `CDSW_APP_PORT` environment variable). The React frontend is served as static files from `app/client/dist` at `/static`, with a catch-all `GET /{path}` handler that falls back to `index.html` for client-side routing.

Source: `app/main.py`, `app/run.py`.

## Authentication

There is no built-in authentication on API routes. CAII (Cloudera AI Inference) endpoints require a bearer token resolved in the following order:

1. `CDP_TOKEN` environment variable.
2. `access_token` field inside `/tmp/jwt` (JSON file).

If neither source provides a token, CAII operations return `401 Unauthorized`. Token validation is performed by `_get_caii_token()` and health-checked by `caii_check()` in `app/core/config.py`.

CORS is configured permissively:

| Parameter | Value |
|---|---|
| `allow_origins` | `["*"]` |
| `allow_credentials` | `True` |
| `allow_methods` | `["*"]` |
| `allow_headers` | `["*"]` |

## Response Envelope

Success responses return direct JSON with no wrapper envelope. Error responses use a consistent envelope:

```json
{"status": "failed", "error": "<message>"}
```

Pydantic validation errors return `422` with FastAPI's default `detail` format (array of field-level error objects).

## Error Codes

The `responses` dict in `app/core/config.py` declares the following status codes. The global middleware in `app/main.py` enforces additional codes at runtime.

| Status Code | Description | Example Error |
|---|---|---|
| 400 | Bad Request | `"Invalid input: No topics provided"` |
| 404 | Not Found | `"Requested resource not found"` |
| 413 | Payload Too Large | `"Total dataset size (X GB) exceeds limit of 10 GB"` |
| 422 | Validation Error | `"Invalid request parameters"` |
| 500 | Internal Server Error | `"Internal server error occurred"` |
| 503 | Service Unavailable | `"The CAII endpoint is downscaled, please try after >15 minutes"` |
| 504 | Gateway Timeout | `"Operation 'endpoint' timed out after N seconds"` |

## Timeout Configuration

Per-endpoint timeouts are resolved by `get_timeout_for_request()` based on the request path. The global middleware wraps `call_next` in `asyncio.wait_for()` with the resolved timeout.

| Endpoint Pattern | Timeout (seconds) |
|---|---|
| `*/generate` | 200 |
| `*/freeform` | 300 |
| `*/create_custom_prompt` | 300 |
| `*/evaluate` | 200 |
| `*/export_results` | 200 |
| `*health*` | 5 |
| `*/upgrade` | 2000 |
| Default | 60 |

Source: `app/main.py` -- `get_timeout_for_request()`.

## NaN Sanitization

Freeform generation responses pass through `deep_sanitize_nans()` before serialisation. This function recursively traverses all nested data structures and applies the following transformations:

| Input Type | Output |
|---|---|
| `float` NaN or Inf | `null` |
| `numpy.floating` NaN or Inf | `null` |
| `numpy.integer` | Python `int` |
| `numpy.floating` (finite) | Python `float` |
| `pandas.Timestamp`, `numpy.datetime64` | ISO 8601 string |
| `numpy.ndarray` | Python `list` (recursively sanitised) |
| `pandas` NA | `null` |

Source: `app/main.py` -- `deep_sanitize_nans()`.

## Middleware Pipeline

Requests are processed in this order:

1. **CORS middleware** -- permissive configuration (see table above).
2. **Global HTTP middleware** -- `global_middleware()` wraps `call_next` in `asyncio.wait_for()` with the per-endpoint timeout, then catches exceptions in a defined hierarchy.

Exception handling order within the global middleware:

| Exception | Status Code | Notes |
|---|---|---|
| `TimeoutError` | 504 | Message includes endpoint name and timeout duration |
| `APIError` | Dynamic (`e.status_code`) | Custom exception from `app/core/exceptions.py` |
| `RuntimeError` | 400 | Treated as bad request |
| `InvalidModelError` | Dynamic (`e.status_code`) | Subclass of `APIError` |
| Catch-all `Exception` | 500 | Generic internal server error |

All error responses use the `{"status": "failed", "error": "..."}` envelope.

## Lifespan Events

The application uses a FastAPI `lifespan` context manager for startup and shutdown.

**Startup:**

1. Creates the `document_upload` directory (`os.makedirs(UPLOAD_DIR, exist_ok=True)`).
2. Runs database migrations via `uv run python run_migrations.py` as a subprocess.

**Shutdown:**

1. Logs `"Application shutting down..."`.

Source: `app/main.py` -- `lifespan()`.
