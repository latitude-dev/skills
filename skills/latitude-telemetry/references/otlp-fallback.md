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

## Workflow

### 1. Confirm OTel SDK presence

Is the language's official OpenTelemetry SDK already a dependency?

- **Yes** → reuse the existing `TracerProvider`. Add an OTLP HTTP exporter to it pointing at the Latitude endpoint. Do not create a second provider.
- **No** → install it. Every supported language has a standard package; check the language's row in the "Per-language pointers" table below.

### 2. Configure the OTLP exporter

Either use the env vars above (preferred — works without code changes for the exporter), or configure the exporter in code with the endpoint and headers. The product docs page has working snippets for Go, Java, Ruby, and .NET.

### 3. Instrument the LLM calls

This is the language-specific step. Three options, in priority order:

1. **Use a community OTel instrumentation** if one exists for the LLM client (see the per-language table). Register it on the same `TracerProvider`.
2. **Wrap the LLM call manually** with a span that follows the `gen_ai.*` semantic conventions (see "Required attributes" below).
3. **Stop and tell the user** if neither is realistic for their stack — for example, a niche PHP HTTP client calling OpenAI directly with no OTel wrapper available.

### 4. Decorate with Latitude attributes

For traces to land usefully in the Latitude UI, set these as standard span attributes (no SDK needed):

| Attribute | Type | Purpose |
| --- | --- | --- |
| `gen_ai.system` | `string` | LLM vendor name (e.g. `"openai"`, `"anthropic"`). Required for the smart UI to recognize the span as an LLM call. |
| `gen_ai.request.model` | `string` | Model name. |
| `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` | `int` | Token counts. |
| `latitude.capture.name` | `string` | Capture name (e.g. `"handle-user-request"`). Optional. |
| `latitude.tags` | `string` (JSON array) | Tags for filtering — `'["production","v2"]'`. Optional. |
| `latitude.metadata` | `string` (JSON object) | Free-form metadata — `'{"requestId":"abc"}'`. Optional. |
| `session.id` | `string` | Group related traces. Optional. |
| `user.id` | `string` | Associate with a user. Optional. |

The full list of LLM semantic conventions: [opentelemetry.io/docs/specs/semconv/gen-ai](https://opentelemetry.io/docs/specs/semconv/gen-ai/).

### 5. Verify before reporting done

Same rule as the main skill: do not declare success because the code compiles. Run the app, trigger one LLM call, confirm the trace appears in the Latitude UI. If nothing arrives, run the curl probe from the docs page first to confirm endpoint connectivity, then check whether the exporter is actually flushing before process exit.

## Per-language pointers

Honest assessment of what is realistic per language. The table below is for **picking which of step 3's three options applies**, not a complete library survey.

| Language | OTel SDK | Realistic LLM instrumentation | Recommendation |
| --- | --- | --- | --- |
| **Go** | `go.opentelemetry.io/otel` (mature) | Sparse community libs; no widely-adopted auto-instrumentation for `openai-go` / `anthropic-sdk-go`. | Manual spans following `gen_ai.*` conventions. |
| **Java** | `io.opentelemetry:*` (mature) | OpenLLMetry has partial Java coverage (e.g. LangChain4j); also community OpenTelemetry instrumentations for some HTTP clients. | Try a community instrumentation first; fall back to manual spans. |
| **Ruby** | `opentelemetry-sdk` (mature) | Sparse; mostly manual. | Manual spans. |
| **.NET** | `OpenTelemetry` packages (mature) | `OpenTelemetry.Instrumentation.*` exists for HTTP/SQL but LLM-specific instrumentations are rare. | Manual spans. |
| **PHP** | `open-telemetry/sdk` | Very thin LLM-instrumentation ecosystem. | Manual spans, or surface this as a gap to the user. |
| **Rust** | `opentelemetry` crate (mature) | Very thin. | Manual spans. |
| **Elixir** | `:opentelemetry` (mature) | Very thin. | Manual spans. |
| **Kotlin / Swift / Other JVM / mobile** | OTel SDK exists but mobile telemetry is unusual for LLM apps. | Manual spans. | Confirm with user that traces should be sent from the mobile/edge process at all (privacy, network); often a server-side proxy is the better boundary. |

If the user's language is not in the table but does have an OTel SDK (e.g. Erlang, Crystal, Haskell), follow the same workflow: standard OTel + OTLP exporter + manual spans with `gen_ai.*` attributes.

## Common mistakes

| Mistake | Why it fails | Fix |
| --- | --- | --- |
| Sending arbitrary, non-LLM spans to the Latitude endpoint | Smart filter does not apply on this path; every span is stored, cluttering the UI | Only export spans that represent LLM operations, or use an `OTEL_TRACES_SAMPLER` / span processor to filter before export |
| Missing `X-Latitude-Project` header | `400 Bad Request` | Include it on every request |
| `Authorization` without the `Bearer ` prefix | `401 Unauthorized` | Use `Bearer <key>` (note the space) |
| No `gen_ai.system` attribute | Span lands but Latitude UI does not classify it as an LLM call | Set `gen_ai.system = "openai"` (or matching vendor) on every LLM span |
| Process exits before exporter flushes | Trace silently dropped | Call the SDK's shutdown / force-flush hook before exit; in serverless, flush before returning |
| Generated TypeScript/Python files into a Go/PHP/etc. codebase | Wrong language; will not compile or run | This skill explicitly does not do that — re-read the "What this skill will and will not do" table at the top |

## When to escalate to the user

Stop and ask the user when:

- The language has no usable OTel SDK at all (extremely rare in 2026, but possible for hobby languages).
- The LLM call goes through a custom HTTP client and the user is unwilling to wrap it in a manual span — there's no other path forward.
- Self-hosted Latitude with an unknown ingest endpoint — the URL above is for production cloud only.

Frame the question concretely: *"I see you're using `<lang>` with `<lib>`. Latitude doesn't have a turnkey path here. The realistic options are: (a) I write manual OTel spans wrapping each LLM call following `gen_ai.*` conventions, (b) you point me at a community instrumentation library you'd like to use, or (c) we file this as a request with Latitude for first-class support. Which do you prefer?"*
