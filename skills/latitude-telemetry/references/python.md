# Python: Latitude Telemetry

Concise reference for `latitude-telemetry`. Always confirm details against the [upstream Python README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/python/README.md).

## Install

**Always install the pre-release (alpha) version.** The stable release lags the API surface this reference describes; without `--pre` the install resolves to a version that is missing functions referenced below.

```bash
pip install --pre latitude-telemetry
```

For other tools, use the equivalent pre-release flag: `uv pip install --prerelease=allow latitude-telemetry`, Poetry `poetry add latitude-telemetry --allow-prereleases`. Do not drop the pre-release flag.

Requires Python 3.11+.

## Path A — Bootstrap (recommended)

```python
from latitude_telemetry import init_latitude

latitude = init_latitude(
    api_key="your-api-key",
    project_slug="your-project-slug",
    instrumentations=["openai"],
)

# ... LLM work ...

latitude["shutdown"]()
```

Use env vars in real apps (`os.environ["LATITUDE_API_KEY"]`, and so on).

> **Note on the return value:** `init_latitude` returns a `TypedDict` with keys `"provider"`, `"flush"`, `"shutdown"`. At runtime that is a plain `dict`, so call them with item-access: `latitude["flush"]()` and `latitude["shutdown"]()`. The README and some docstrings show `latitude.shutdown()` (attribute access) — that fails at runtime with `AttributeError`. The example test files (`packages/telemetry/python/examples/`) use `latitude["flush"]()` and are the source of truth.

## Path B — Existing OpenTelemetry (advanced)

```python
from opentelemetry.sdk.trace import TracerProvider
from latitude_telemetry import LatitudeSpanProcessor, register_latitude_instrumentations

provider = TracerProvider()
provider.add_span_processor(LatitudeSpanProcessor("api-key", "project-slug"))
# add other processors / exporters as needed
provider.register()

register_latitude_instrumentations(
    instrumentations=["openai"],
    tracer_provider=provider,
)
```

`LatitudeSpanProcessor` exports to Latitude; you still need LLM instrumentations (via `register_latitude_instrumentations` or your own compatible instrumentation) to create spans.

## Optional context: `capture()`

`capture` supports two forms in Python. Pick whichever fits the call site; they both attach the same context.

### Decorator form (preferred for whole functions)

This is the primary pattern in the upstream example files (`examples/test_openai.py`, etc.).

```python
from latitude_telemetry import init_latitude, capture

latitude = init_latitude(
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
    },
)
def handle_user_request(user_message: str) -> str:
    # LLM call inside this function gets the context above
    response = client.chat.completions.create(...)
    return response.choices[0].message.content

result = handle_user_request("hello")
latitude["flush"]()
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

latitude["shutdown"]()
```

`ContextOptions` keys: `name` (optional), `user_id`, `session_id`, `tags`, `metadata`. Same merge rules as TypeScript: tags merge+dedupe, metadata shallow-merge, ids last-write-wins.

## Public surface

```python
from latitude_telemetry import (
    init_latitude,
    LatitudeSpanProcessor,
    capture,
    register_latitude_instrumentations,
)
```

## Common pitfalls

| Symptom | Things to verify |
| ------- | ---------------- |
| No spans in Latitude | Credentials; instrumentations registered; smart filter |
| Missing spans at exit | `flush()` / `shutdown()` / `force_flush()` on provider |
| `capture()` seems empty | Active LLM instrumentation must create spans inside the callback |
