# Python: Latitude Telemetry

Concise reference for `latitude-telemetry`. Always confirm details against the [upstream Python README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/python/README.md).

## Install

```bash
pip install latitude-telemetry
```

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

latitude.shutdown()
```

Use env vars in real apps (`os.environ["LATITUDE_API_KEY"]`, and so on).

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

```python
from latitude_telemetry import init_latitude, capture

latitude = init_latitude(
    api_key="your-api-key",
    project_slug="your-project-slug",
    instrumentations=["openai"],
)

capture(
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

Use the dict keys documented in the upstream README (`user_id`, `session_id`, `tags`, `metadata`, optional `name`).

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
