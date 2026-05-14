# Python: Latitude Telemetry

Concise reference for `latitude-telemetry`. Always confirm details against the [upstream Python README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/python/README.md).

## Install

**Always look up the current pre-release and pin to it.** The SDK is on the alpha / beta channel; `--pre` floats by default, so re-running the install can land on a different version with a different API surface. Look up once, pin exact.

```bash
# List all versions; the first non-stable entry is the current pre-release.
pip index versions latitude-telemetry --pre

# Pin the install to the exact version you found.
pip install "latitude-telemetry==3.0.0a10"     # substitute the actual version
# Or with uv:
uv pip install "latitude-telemetry==3.0.0a10"
# Or with Poetry:
poetry add "latitude-telemetry==3.0.0a10" --allow-prereleases
```

Confirm the lockfile (`uv.lock`, `poetry.lock`, `requirements.txt`, etc.) captures the pin.

Requires Python 3.11+.

## Path A — Bootstrap (recommended)

`Latitude(...)` is the canonical entry point. It auto-detects an existing OpenTelemetry tracer provider and attaches its span processor to it; if none exists, it creates and registers one. Instrumentations register synchronously in `__init__` — there is no `ready` attribute and nothing to await (TypeScript has one, Python does not).

```python
import os
from latitude_telemetry import Latitude, capture

latitude = Latitude(
    api_key=os.environ["LATITUDE_API_KEY"],
    project_slug=os.environ["LATITUDE_PROJECT_SLUG"],
    instrumentations=["openai"],
)

# ... LLM work ...

latitude.shutdown()
```

Real attributes and methods — no dict access:

- `latitude.provider` — the `TracerProvider` instance.
- `latitude.flush()` — force a flush of pending spans (use in serverless handlers before returning).
- `latitude.shutdown()` — flush and tear down. Call before a CLI / script exits.

### `project_slug` is optional on the constructor

If your app emits to several Latitude projects (multi-agent, multi-feature), omit `project_slug` from the constructor and pass it on each `capture()` instead. See [project-scoping.md](project-scoping.md) for the full pattern and precedence rules. For single-project apps, keep `project_slug` in the constructor.

### Legacy `init_latitude`

`init_latitude(...)` still exists and returns a `TypedDict` with `provider`, `flush`, `shutdown` keys — that's the old shape. New installs should use `Latitude(...)` (the class). If you're reading an existing codebase that calls `init_latitude`, leave it alone; the wrapper is supported. If you see `latitude.shutdown()` (attribute access) **on the dict return value**, that's a real bug — flag it. If you see `latitude["shutdown"]()` (item access) **on a `Latitude(...)` class instance**, that's also a bug — the class uses real methods.

## Path B — Existing OpenTelemetry (advanced)

Pass an existing `TracerProvider` to the constructor:

```python
from opentelemetry.sdk.trace import TracerProvider
from latitude_telemetry import Latitude

provider = TracerProvider()
# ...attach your existing processors/exporters to `provider`...

latitude = Latitude(
    api_key="api-key",
    project_slug="project-slug",
    instrumentations=["openai"],
    tracer_provider=provider,
)
```

The constructor attaches `LatitudeSpanProcessor` to the provider you passed; your existing processors continue to receive every span. If you skip `tracer_provider`, the SDK auto-detects a globally-registered provider — same outcome.

For the rare case where you want to manage everything yourself, the lower-level primitives are still exported:

```python
from latitude_telemetry import LatitudeSpanProcessor, register_latitude_instrumentations

provider.add_span_processor(LatitudeSpanProcessor(api_key="...", project_slug="..."))

register_latitude_instrumentations(
    instrumentations=["openai"],
    tracer_provider=provider,
)
```

Use this only when the class's auto-detection genuinely doesn't fit. For everything else, `Latitude(...)` is shorter and less error-prone.

## Optional context: `capture()`

`capture` supports two forms in Python. Pick whichever fits the call site; they both attach the same context.

### Decorator form (preferred for whole functions)

This is the primary pattern in the upstream example files (`examples/test_openai.py`, etc.).

```python
from latitude_telemetry import Latitude, capture

latitude = Latitude(
    api_key="your-api-key",
    project_slug="your-project-slug",
    instrumentations=["openai"],
)

@capture(
    "handle-user-request",
    {
        "tags": ["production", "v2-agent"],
        "session_id": "session_abc",
        "user_id": "user_123",
        "metadata": {"request_id": "req-xyz"},
        # Optional — routes this capture to a different Latitude project. See project-scoping.md.
        # "project_slug": "evaluation-runs",
    },
)
def handle_user_request(user_message: str) -> str:
    # LLM call inside this function gets the context above
    response = client.chat.completions.create(...)
    return response.choices[0].message.content

result = handle_user_request("hello")
latitude.flush()
```

Async functions are supported the same way (`@capture(...)` over `async def`).

### Callback form (for inline code)

```python
result = capture(
    "handle-user-request",
    lambda: agent.process(user_message),
    {
        "user_id": "user_123",
        "session_id": "session_abc",
        "tags": ["production", "v2-agent"],
        "metadata": {"request_id": "req-xyz"},
    },
)

latitude.shutdown()
```

`ContextOptions` keys: `name` (optional), `user_id`, `session_id`, `tags`, `metadata`, `project_slug` (optional). Same merge rules as TypeScript: tags merge+dedupe, metadata shallow-merge, ids last-write-wins.

## Public surface

```python
from latitude_telemetry import (
    Latitude,
    LatitudeSpanProcessor,
    capture,
    register_latitude_instrumentations,
)
```

`init_latitude` is still exported for backwards compatibility but returns the old `TypedDict` shape; prefer the class form for new code.

## Supported instrumentation identifiers

The pattern is the same as TypeScript: pick the identifier that matches the SDK in use and pass it in `instrumentations=[...]`. There is no special wiring per framework — the identifier is the entire integration.

```
"openai" | "openai-agents" | "anthropic" | "bedrock" | "cohere"
"langchain" | "llamaindex" | "togetherai" | "vertexai" | "aiplatform"
```

| If the app uses | Add to `instrumentations` |
| --- | --- |
| `openai` (chat completions / responses API) | `"openai"` |
| `openai-agents` (the Agents SDK Python package) | `"openai-agents"` — dedicated instrumentation, do **not** use `"openai"` here. See [docs.latitude.so/telemetry/frameworks/openai-agents](https://docs.latitude.so/telemetry/frameworks/openai-agents) |
| `anthropic` | `"anthropic"` |
| `boto3` Bedrock client | `"bedrock"` |
| `cohere` | `"cohere"` |
| `together` | `"togetherai"` |
| `google-cloud-aiplatform` | `"aiplatform"` (or `"vertexai"` for the Vertex AI client) |
| `langchain-core`, `langchain` | `"langchain"` |
| `llama-index` | `"llamaindex"` |

For the OpenAI Agents Python install (substituting the version you got from the Install section):

```bash
pip install "latitude-telemetry==3.0.0a10" openai-agents
```

```python
latitude = Latitude(
    api_key="your-api-key",
    project_slug="your-project-slug",
    instrumentations=["openai-agents"],
)
```

## Common pitfalls

| Symptom | Things to verify |
| ------- | ---------------- |
| No spans in Latitude | Credentials; instrumentations registered; smart filter; the pinned version actually matches what the install code expects (re-run the lookup from the Install section to check for drift) |
| Missing spans at exit | `latitude.flush()` / `latitude.shutdown()` (methods on the class, not dict items) |
| `capture()` seems empty | Active LLM instrumentation must create spans inside the callback |
| `AttributeError: 'dict' object has no attribute 'shutdown'` | Code is using `init_latitude(...)` (legacy dict return) but calling `latitude.shutdown()` — switch to `Latitude(...)` (the class) so attribute access works, or call `latitude["shutdown"]()` if you must keep the legacy form |
