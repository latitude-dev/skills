# OTLP fallback (non-TypeScript / non-Python)

Use this reference when the host language is **not** TypeScript or Python. Latitude does not ship a first-class SDK for other languages, but the ingest endpoint speaks standard **OTLP over HTTP**, so any application with an OpenTelemetry SDK (Go, Java, Ruby, .NET, PHP, Rust, Elixir, Kotlin, Swift, …) can send traces.

The product docs at [docs.latitude.so/telemetry/otel-exporter](https://docs.latitude.so/telemetry/otel-exporter) are the source of truth — fetch that page when in doubt.

## What this skill will and will not do here

| ✅ Will do | ❌ Will not do |
| --- | --- |
| Configure the language's standard OTel SDK with an OTLP HTTP exporter pointed at Latitude. | Generate TypeScript or Python files into a non-TS/Python codebase. |
| Emit spans following `gen_ai.*` semantic conventions for LLM calls. | Invent a "Latitude SDK for $LANG" — there isn't one. |
| Decorate spans with Latitude-specific attributes (`user.id`, `session.id`, `latitude.tags`, `latitude.metadata`). | Promise auto-instrumentation for an LLM client library that has no community OTel instrumentation. |
| Verify the endpoint with `curl` before declaring done. | Generate a working "agent run" example that the agent cannot validate end-to-end. |

If the language has no viable LLM-instrumentation ecosystem **and** the user is not willing to write manual spans, **say so**. Do not fake it. Direct the user to `https://docs.latitude.so/telemetry/otel-exporter` and offer to file a request with Latitude for first-class support.

## Endpoint and required headers

| | Value |
| --- | --- |
| URL | `https://ingest.latitude.so/v1/traces` (production default — confirm with user if self-hosted) |
| Method | `POST` |
| `Authorization` | `Bearer <LATITUDE_API_KEY>` |
| `X-Latitude-Project` | `<LATITUDE_PROJECT_SLUG>` |
| `Content-Type` | `application/json` or `application/x-protobuf` |

Returns `202` on success. The smart filter does **not** apply on this path — every span sent is ingested. Do not export non-LLM spans you don't want stored.

Most OTel SDKs respect these env vars, so you can often skip code changes entirely:

```bash
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="https://ingest.latitude.so/v1/traces"
export OTEL_EXPORTER_OTLP_TRACES_HEADERS="Authorization=Bearer <key>,X-Latitude-Project=<slug>"
```

## Pick the language reference

Open the matching file for concrete install commands, OTel setup, and a manual LLM-span template:

| Language | Reference | LLM-instrumentation ecosystem |
| --- | --- | --- |
| Go | [otlp-go.md](otlp-go.md) | Sparse — manual spans realistic |
| Java | [otlp-java.md](otlp-java.md) | Partial OpenLLMetry coverage; check community libs first, manual otherwise |
| Ruby | [otlp-ruby.md](otlp-ruby.md) | Sparse — manual spans |
| .NET | [otlp-dotnet.md](otlp-dotnet.md) | Sparse — manual spans |
| PHP | [otlp-php.md](otlp-php.md) | Very thin — manual spans, plus FPM flush discipline |
| Rust, Elixir, Kotlin, Swift, anything else | No dedicated reference — see [Other languages](#other-languages) below | Very thin or absent |

Each per-language reference covers: install command, minimum OTel setup with the Latitude headers, a manual LLM-span template wrapping an OpenAI-style call with `gen_ai.*` attributes, the flush-before-exit pattern for that runtime, and language-specific common pitfalls.

## Latitude UI attributes

Every per-language template sets these. Required for the trace to be useful in the Latitude UI:

| Attribute | Type | Purpose |
| --- | --- | --- |
| `gen_ai.system` | `string` | LLM vendor name (`"openai"`, `"anthropic"`, `"bedrock"`, …). Required for the UI to recognize the span as an LLM call. |
| `gen_ai.request.model` | `string` | Model name. |
| `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` | `int` | Token counts. |
| `gen_ai.response.model` | `string` | Actual model returned (often equal to request, but not always). |

Optional Latitude-specific attributes for filtering and grouping:

| Attribute | Type | Purpose |
| --- | --- | --- |
| `latitude.capture.name` | `string` | Capture name (e.g. `"handle-user-request"`). |
| `latitude.tags` | `string` (JSON array) | Tags for filtering — `'["production","v2"]'`. |
| `latitude.metadata` | `string` (JSON object) | Free-form metadata — `'{"requestId":"abc"}'`. |
| `session.id` | `string` | Group related traces. |
| `user.id` | `string` | Associate with a user. |

Full LLM semantic conventions: [opentelemetry.io/docs/specs/semconv/gen-ai](https://opentelemetry.io/docs/specs/semconv/gen-ai/).

## Other languages

If the language is not in the table above (Rust, Elixir, Kotlin, Swift, Crystal, Haskell, …):

1. Confirm the language has an official OpenTelemetry SDK at [opentelemetry.io/docs/languages](https://opentelemetry.io/docs/languages/). Almost every mainstream language does.
2. Follow the same recipe used in the per-language references: standard OTel SDK + OTLP HTTP exporter to `https://ingest.latitude.so/v1/traces` with `Authorization: Bearer <key>` and `X-Latitude-Project: <slug>` headers.
3. Wrap LLM calls with manual spans setting `gen_ai.system`, `gen_ai.request.model`, and the token-usage attributes from the table above.
4. If the language has no usable OTel SDK or no realistic way to wrap LLM calls, escalate to the user using the script in [When to escalate](#when-to-escalate-to-the-user).

Pick a per-language reference from the table that uses the closest patterns (Rust ≈ Go for tracer/span lifetime; Elixir ≈ Ruby for block-style; Kotlin ≈ Java) as a structural template, then adapt to the target language's idioms.

## Verify with curl

Before instrumenting code, prove the endpoint works for the user's API key and project:

```bash
curl -X POST https://ingest.latitude.so/v1/traces \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "X-Latitude-Project: YOUR_PROJECT_SLUG" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": { "attributes": [{ "key": "service.name", "value": { "stringValue": "curl-test" } }] },
      "scopeSpans": [{
        "scope": { "name": "manual-test" },
        "spans": [{
          "traceId": "00000000000000000000000000000001",
          "spanId": "0000000000000001",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1700000000000000000",
          "endTimeUnixNano": "1700000001000000000",
          "attributes": [{ "key": "gen_ai.system", "value": { "stringValue": "openai" } }]
        }]
      }]
    }]
  }'
```

A `202` response with `{}` means the endpoint accepted the payload. Confirm the trace appears in the Latitude UI before continuing.

## Common mistakes (universal)

Language-specific pitfalls live in each per-language reference. The following bite on every language:

| Mistake | Why it fails | Fix |
| --- | --- | --- |
| Sending arbitrary, non-LLM spans | Smart filter does not apply on this path; every span is stored, cluttering the UI | Only export spans that represent LLM operations, or filter via a span processor before export |
| Missing `X-Latitude-Project` header | `400 Bad Request` | Include it on every request |
| `Authorization` without the `Bearer ` prefix | `401 Unauthorized` | Use `Bearer <key>` (note the space) |
| No `gen_ai.system` attribute | Span lands but Latitude UI does not classify it as an LLM call | Set `gen_ai.system = "openai"` (or matching vendor) on every LLM span |
| Process exits before exporter flushes | Trace silently dropped | Call the SDK's shutdown / force-flush hook before exit; in serverless, flush before returning |
| Generated TypeScript/Python files into a Go/PHP/etc. codebase | Wrong language; will not compile or run | Re-read [What this skill will and will not do](#what-this-skill-will-and-will-not-do-here) |

## When to escalate to the user

Stop and ask the user when:

- The language has no usable OTel SDK at all (rare in 2026, but possible for hobby languages).
- The LLM call goes through a custom HTTP client and the user is unwilling to wrap it in a manual span — there's no other path forward.
- Self-hosted Latitude with an unknown ingest endpoint — the URL above is for production cloud only.

Frame the question concretely: *"I see you're using `<lang>` with `<lib>`. Latitude doesn't have a turnkey path here. The realistic options are: (a) I write manual OTel spans wrapping each LLM call following `gen_ai.*` conventions, (b) you point me at a community instrumentation library you'd like to use, or (c) we file this as a request with Latitude for first-class support. Which do you prefer?"*
