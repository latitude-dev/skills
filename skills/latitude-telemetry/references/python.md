# Python: Latitude Telemetry

Concise reference for `latitude-telemetry`. Always confirm details against the [upstream Python README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/python/README.md).

## Install

**Always look up the current pre-release and pin to it.** The SDK is on the alpha / beta channel; `--pre` floats by default, so re-running the install can land on a different version with a different API surface. Look up once, pin exact.

```bash
# List all versions; the first non-stable entry is the current pre-release.
pip index versions latitude-telemetry --pre

# Pin the install to the exact version you found.
pip install "latitude-telemetry==3.0.0a7"     # substitute the actual version
# Or with uv:
uv pip install "latitude-telemetry==3.0.0a7"
# Or with Poetry:
poetry add "latitude-telemetry==3.0.0a7" --allow-prereleases
```

Confirm the lockfile (`uv.lock`, `poetry.lock`, `requirements.txt`, etc.) captures the pin.

Requires Python 3.11+.

## Path A — Bootstrap (recommended)

`Latitude(...)` is the canonical entry point. It auto-detects an existing OpenTelemetry tracer provider and attaches its span processor to it; if none exists, it creates and registers one. Instrumentations register synchronously in `__init__` — there is no `ready` attribute and nothing to await (TypeScript has one, Python does not).

```python
import os

import openai

from latitude_telemetry import Latitude, capture

latitude = Latitude(
    api_key=os.environ["LATITUDE_API_KEY"],
    project=os.environ["LATITUDE_PROJECT_SLUG"],
    instrumentations={"openai": openai},
)

# ... LLM work ...

latitude.shutdown()
```

`instrumentations` is a dict mapping integration name (`openai`, `anthropic`, …) to the LLM SDK module the consumer imports. Passing the user's own module sidesteps a class of import-cache bugs where the SDK could patch a different module instance than the app loads. Anything other than a plain dict (including the legacy list-of-strings form) raises `TypeError` at register time.

Real attributes and methods — no dict access:

- `latitude.provider` — the `TracerProvider` instance.
- `latitude.flush()` — force a flush of pending spans (use in serverless handlers before returning).
- `latitude.shutdown()` — flush and tear down. Call before a CLI / script exits.

### `project` is optional on the constructor

If your app emits to several Latitude projects (multi-agent, multi-feature), omit `project` from the constructor and pass it on each `capture()` instead. See [project-scoping.md](project-scoping.md) for the full pattern and precedence rules. For single-project apps, keep `project` in the constructor.

> The constructor option was renamed from `project_slug` to `project` in `latitude-telemetry` ≥ `3.0.0a8`. `project_slug` still works but logs a one-time deprecation warning; new code should use `project`.

### Legacy `init_latitude`

`init_latitude(...)` still exists and returns a `TypedDict` with `provider`, `flush`, `shutdown` keys — that's the old shape. New installs should use `Latitude(...)` (the class). If you're reading an existing codebase that calls `init_latitude`, leave it alone; the wrapper is supported. If you see `latitude.shutdown()` (attribute access) **on the dict return value**, that's a real bug — flag it. If you see `latitude["shutdown"]()` (item access) **on a `Latitude(...)` class instance**, that's also a bug — the class uses real methods.

### Legacy `instrumentations=["openai"]` is removed — and you MUST force-migrate if you see it

The list-of-strings form is removed in `3.0.0a7`+ with **no fallback**. Any non-dict value — including the old string list — **raises `TypeError`** at register time: `[Latitude] instrumentations must be a dict mapping integration names to LLM SDK modules (e.g. {'openai': openai, 'anthropic': anthropic})`.

When you see `@latitude-data/telemetry`'s Python package (`latitude-telemetry`) installed:

1. **Read the installed version** from `pyproject.toml` + lockfile. If `< 3.0.0a7`, you are mid-breaking-upgrade.
2. **Grep for every legacy call site:** `rg -n 'instrumentations=\[' .` — every match must be rewritten.
3. **Rewrite each match.** Add `import <pkg>` at the top of the file (the module-level import, not `from <pkg> import X`) and pass that module into the dict:

   ```diff
   - from latitude_telemetry import Latitude
   + import openai
   + import anthropic
   + from latitude_telemetry import Latitude

     latitude = Latitude(
         api_key=os.environ["LATITUDE_API_KEY"],
         project=os.environ["LATITUDE_PROJECT_SLUG"],
   -     instrumentations=["openai", "anthropic"],
   +     instrumentations={"openai": openai, "anthropic": anthropic},
     )
   ```

4. **Bump the version pin** to the latest alpha returned by `pip index versions latitude-telemetry --pre`, and update the lockfile.
5. **Run the app once** to confirm the bootstrap doesn't raise `TypeError: [Latitude] instrumentations must be a dict mapping…`.

This is the Python counterpart to the TypeScript SDK's object-form migration — both SDKs now share the same `instrumentations` shape.

## Path B — Existing OpenTelemetry (advanced)

Pass an existing `TracerProvider` to the constructor:

```python
import openai
from opentelemetry.sdk.trace import TracerProvider
from latitude_telemetry import Latitude

provider = TracerProvider()
# ...attach your existing processors/exporters to `provider`...

latitude = Latitude(
    api_key="api-key",
    project="project-slug",
    instrumentations={"openai": openai},
    tracer_provider=provider,
)
```

The constructor attaches `LatitudeSpanProcessor` to the provider you passed; your existing processors continue to receive every span. If you skip `tracer_provider`, the SDK auto-detects a globally-registered provider — same outcome.

For the rare case where you want to manage everything yourself, the lower-level primitives are still exported:

```python
import openai
from latitude_telemetry import LatitudeSpanProcessor, register_latitude_instrumentations

provider.add_span_processor(LatitudeSpanProcessor(api_key="...", project="..."))

register_latitude_instrumentations(
    instrumentations={"openai": openai},
    tracer_provider=provider,
)
```

Use this only when the class's auto-detection genuinely doesn't fit. For everything else, `Latitude(...)` is shorter and less error-prone.

## Optional context: `capture()`

`capture` supports two forms in Python. Pick whichever fits the call site; they both attach the same context.

### Decorator form (preferred for whole functions)

This is the primary pattern in the upstream example files (`examples/test_openai.py`, etc.).

```python
import openai

from latitude_telemetry import Latitude, capture

latitude = Latitude(
    api_key="your-api-key",
    project="your-project-slug",
    instrumentations={"openai": openai},
)

@capture(
    "handle-user-request",
    {
        "tags": ["production", "v2-agent"],
        "session_id": "session_abc",
        "user_id": "user_123",
        "metadata": {"request_id": "req-xyz"},
        # Optional — routes this capture to a different Latitude project. See project-scoping.md.
        # "project": "evaluation-runs",
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

`ContextOptions` keys: `name` (optional), `user_id`, `session_id`, `tags`, `metadata`, `project` (optional; legacy alias `project_slug` still works with a deprecation warning). Same merge rules as TypeScript: tags merge+dedupe, metadata shallow-merge, ids last-write-wins.

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

Set the integration's key on the `instrumentations` dict to the LLM SDK module the consumer imports.

```
# Shared with TypeScript:
"openai" | "openai-agents" | "anthropic" | "bedrock" | "cohere"
"langchain" | "llamaindex" | "togetherai" | "vertexai" | "aiplatform"

# Python-only (the OpenLLMetry Python ecosystem ships more pre-built instrumentors):
"aleph_alpha" | "crewai" | "dspy" | "google_generativeai" | "groq"
"haystack" | "litellm" | "mistralai" | "ollama" | "replicate"
"sagemaker" | "transformers" | "watsonx"
```

| If the app uses | What to pass on `instrumentations` |
| --- | --- |
| `openai` (chat completions / responses API) | `import openai` → `{"openai": openai}` |
| `openai-agents` (the Agents SDK Python package) | `import agents` → `{"openai-agents": agents}` — dedicated instrumentation, do **not** use `"openai"` for this. See [docs.latitude.so/telemetry/frameworks/openai-agents](https://docs.latitude.so/telemetry/frameworks/openai-agents) |
| `anthropic` | `import anthropic` → `{"anthropic": anthropic}` |
| `boto3` Bedrock client | `import boto3` → `{"bedrock": boto3}` |
| `cohere` | `import cohere` → `{"cohere": cohere}` |
| `together` | `import together` → `{"togetherai": together}` |
| `google-cloud-aiplatform` | `import google.cloud.aiplatform` → `{"aiplatform": <module>}` (or `import vertexai` → `{"vertexai": vertexai}` for the Vertex AI client) |
| `langchain-core`, `langchain` | `import langchain_core` → `{"langchain": langchain_core}` |
| `llama-index` | `import llama_index` → `{"llamaindex": llama_index}` |
| `aleph-alpha-client` | `import aleph_alpha_client` → `{"aleph_alpha": aleph_alpha_client}` |
| `crewai` | `import crewai` → `{"crewai": crewai}` |
| `dspy-ai` | `import dspy` → `{"dspy": dspy}` |
| Gemini (`google-generativeai`) | `from google import genai` → `{"google_generativeai": genai}` |
| `groq` | `import groq` → `{"groq": groq}` |
| `haystack-ai` | `import haystack` → `{"haystack": haystack}` |
| `litellm` | `import litellm` → `{"litellm": litellm}` |
| `mistralai` | `import mistralai` → `{"mistralai": mistralai}` |
| `ollama` | `import ollama` → `{"ollama": ollama}` |
| `replicate` | `import replicate` → `{"replicate": replicate}` |
| SageMaker via `boto3` | `import boto3` → `{"sagemaker": boto3}` |
| `transformers` (Hugging Face) | `import transformers` → `{"transformers": transformers}` |
| `ibm-watson-machine-learning` | `import ibm_watsonx_ai` → `{"watsonx": ibm_watsonx_ai}` |

For the OpenAI Agents Python install (substituting the version you got from the Install section):

```bash
pip install "latitude-telemetry==3.0.0a7" openai-agents
```

```python
import agents

latitude = Latitude(
    api_key="your-api-key",
    project="your-project-slug",
    instrumentations={"openai-agents": agents},
)
```

## Common pitfalls

| Symptom | Things to verify |
| ------- | ---------------- |
| `TypeError: [Latitude] instrumentations must be a dict mapping…` at startup | Codebase is on the removed list-of-strings form. Migrate to `{"openai": openai, …}` — see "Legacy `instrumentations=["openai"]` is removed" above. |
| `TypeError: [Latitude] instrumentations: unknown integration "..."` | The dict key isn't in the supported set. Check spelling against the table above (`aleph_alpha`, not `aleph-alpha`; `mistralai`, not `mistral`). |
| No spans in Latitude | Credentials; instrumentations registered; smart filter; the pinned version actually matches what the install code expects (re-run the lookup from the Install section to check for drift) |
| Missing spans at exit | `latitude.flush()` / `latitude.shutdown()` (methods on the class, not dict items) |
| `capture()` seems empty | Active LLM instrumentation must create spans inside the callback |
| `AttributeError: 'dict' object has no attribute 'shutdown'` | Code is using `init_latitude(...)` (legacy dict return) but calling `latitude.shutdown()` — switch to `Latitude(...)` (the class) so attribute access works, or call `latitude["shutdown"]()` if you must keep the legacy form |
