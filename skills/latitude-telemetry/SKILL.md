---
name: latitude-telemetry
description: Install or audit Latitude LLM observability — sends traces from LLM SDKs (OpenAI, Anthropic, Bedrock, Cohere, TogetherAI, Vertex AI, Google AI Platform, OpenAI Agents, Vercel AI SDK, Mastra, LangChain, LlamaIndex) to a Latitude project. TypeScript via `@latitude-data/telemetry`, Python via `latitude-telemetry`, and any other OpenTelemetry-supported language (Go, Java, Ruby, .NET, PHP, Rust, Elixir, …) via direct OTLP HTTP. Use when the user asks to add Latitude tracing, wire Latitude into an existing OpenTelemetry setup, fix missing traces in Latitude, or audit an existing integration. Covers codebase discovery (existing OTel, conflicting LLM-observability vendors, which LLM SDKs are in use, where LLM calls happen), `initLatitude` / `init_latitude` bootstrap, advanced setup with `LatitudeSpanProcessor` and `registerLatitudeInstrumentations`, optional `capture()` for user/session/tags context, env vars (`LATITUDE_API_KEY`, `LATITUDE_PROJECT_SLUG`), and an OTLP fallback path for non-TS/Python codebases.
---

# Latitude Telemetry

Install Latitude LLM observability into a user's codebase **correctly the first time**. Most failed installs share the same root cause: telemetry was added in a place that does not actually run an LLM call, so no traces appear. This skill guides you through codebase discovery before writing any code.

## Two rules you must not break

**Rule 1 — `capture()` does NOT create spans.** It only attaches user/session/tags/metadata to spans that auto-instrumentation creates from inside the callback.

**Rule 2 — Always `await latitude.ready` (TS) / `latitude["ready"]` is implicit (Py) before the first LLM call.** `initLatitude` returns immediately and registers patches in the background. If the first LLM call fires before `ready` resolves, the patch may not have hooked the SDK yet and the trace is silently lost. Past installs by this skill have shipped without the await and produced empty trace lists. Make this the first line after `initLatitude`.

Concretely for Rule 1:

- ✅ `capture("handle-chat", () => openai.chat.completions.create(...))` — the OpenAI auto-instrumentation creates the span; `capture` decorates it.
- ❌ `capture("compute-prompt", () => buildPromptString(...))` — no LLM call inside, no span, no trace.
- ❌ Wrapping the whole HTTP server `register` callback — telemetry has not started yet.

If the user reports "no traces appear," 90% of the time the `capture()` callback does not invoke an auto-instrumented LLM SDK, **or** Rule 2 was skipped. Verify both before debugging anything else.

## Workflow

Run these steps in order. Do not skip discovery — that is what makes this skill different from "read a README and paste a snippet."

### Step 1 — Confirm credentials exist and reach the project

Latitude needs two values; a third is optional and only matters for self-hosted or local-dev users:

| Variable | Required | Where to find it |
| --- | --- | --- |
| `LATITUDE_API_KEY` | yes | latitude.so → workspace → Settings → API keys → New key |
| `LATITUDE_PROJECT_SLUG` | yes | The slug in the project URL after `/projects/`, or project Settings → General |
| `LATITUDE_TELEMETRY_URL` | no | Override the OTLP ingest URL. **Defaults to `https://ingest.latitude.so`** in production, `http://localhost:3002` in dev. Leave it unset when pointing at production Latitude — set it only for self-hosted instances or non-default endpoints. |

#### 1a. Look in the repo first

Before asking the user, search for already-configured credentials in this order:

1. `.env`, `.env.local`, `.env.development`, `.env.production`, `.env.example`
2. Host-specific secrets files (`fly.toml`, `vercel.json`, IaC manifests, Helm `values.yaml`, GitHub/GitLab CI variables in workflow files)
3. Existing `process.env.LATITUDE_*` / `os.environ["LATITUDE_*"]` references in code

If both values are already wired up, jump to 1c. If exactly one is missing, ask the user only for that one — don't ask for both blindly.

#### 1b. How to ask, if missing

Quote the exact location so the user doesn't have to hunt:

> "I need two values to wire this up. The API key is at **latitude.so → workspace Settings → API keys** (sign up at latitude.so if you don't have an account yet). The project slug is the slug in the project URL after `/projects/`, or in **project Settings → General**. Paste them here, or tell me which `.env` file to add them to.
>
> Are you connecting against production Latitude or a self-hosted / local instance? If production, you can ignore the next part — the SDK defaults to `https://ingest.latitude.so`. If self-hosted or running Latitude locally, also tell me the ingest URL so I can set `LATITUDE_TELEMETRY_URL` to that value."

Never hardcode the key. Load from `process.env` / `os.environ`. If the user pastes a key, write it to `.env`, add a `.env.example` placeholder for collaborators, and confirm `.env` is in `.gitignore`.

For the ingest URL: only set `LATITUDE_TELEMETRY_URL` if the user explicitly says they're on self-hosted or a non-production endpoint. If they don't mention it or say "production", leave it unset — the SDK already points at `https://ingest.latitude.so`.

#### 1c. Verify credentials reach the project before writing any code

Run the curl probe in [references/otlp-fallback.md](references/otlp-fallback.md#verify-with-curl) — it works regardless of language and tells you in one shot:

- `202` → credentials valid, project exists. Continue.
- `401` → bad API key. Re-check at latitude.so → Settings → API keys.
- `400` with a missing-project message → wrong project slug or wrong header name.

Do not write SDK code until the probe returns `202`. This catches the LAT-558 class of bug ("code looks fine, no traces appear") at the credential layer instead of after a full implementation.

### Step 2 — Discover the codebase

Before deciding which install path to use, gather these facts. Read code and grep — only ask the user when the codebase truly cannot answer.

#### 2a. Language gate (do this first)

Detect the host language **before** anything else. Latitude ships first-class SDKs only for TypeScript and Python; everything else uses a different install path.

| Language | Path |
| --- | --- |
| TypeScript / JavaScript / Node | Continue with this skill — `@latitude-data/telemetry` |
| Python | Continue with this skill — `latitude-telemetry` |
| Go, Java, Ruby, .NET, PHP, Rust, Elixir, Kotlin, Swift, anything else | **Stop and switch to [references/otlp-fallback.md](references/otlp-fallback.md)** — there is no Latitude SDK for this language; install standard OpenTelemetry pointed at the OTLP HTTP endpoint instead |

If the codebase mixes languages (e.g. Python backend + Go data pipeline), instrument each independently using the right path per language. Do not bridge them through one SDK.

Once the language is confirmed as TS or Python, also note:

- TypeScript / Node — Plain Node, Next.js, NestJS, Hono, Fastify, Express, CLI script?
- Python — FastAPI, Flask, Django, Celery worker, plain script, notebook?
- Process shape — long-running server, cron, serverless function, one-shot CLI, batch job?

#### 2b. Existing OpenTelemetry instrumentation

Grep for these. If any are present, the install path changes from bootstrap to advanced.

| Search term | What it indicates |
| --- | --- |
| `@opentelemetry/sdk-node`, `@opentelemetry/sdk-trace-node`, `NodeSDK`, `NodeTracerProvider` | Existing TS OTel SDK |
| `opentelemetry.sdk.trace`, `TracerProvider()` | Existing Python OTel SDK |
| `OTEL_EXPORTER_*`, `OTEL_SERVICE_NAME`, `OTEL_TRACES_*` env vars | Configured OTel exporter |
| `instrumentation.ts`, `instrumentation.js` (Next.js) | OTel hook already wired |

If found, plan the **advanced** path: add `LatitudeSpanProcessor` to the existing `TracerProvider` and call `registerLatitudeInstrumentations` / `register_latitude_instrumentations` against that same provider. Do not create a second provider.

#### 2c. Conflicting LLM observability vendors

Grep for these. If any are present, do not silently overlap — ask the user whether they want to keep, replace, or run in parallel.

```
langfuse  langsmith  traceloop  helicone  arize  phoenix  openllmetry  weights-and-biases  wandb
```

Special cases:

- **Traceloop OpenLLMetry**: shares the same underlying instrumentations Latitude uses. Two registrations of the same instrumentation can cause double-counting. Use the **advanced** path with one shared provider.
- **Langfuse / LangSmith**: independent vendors. They can coexist with Latitude through OTel, but the user should explicitly want both.

#### 2d. Which LLM SDKs are in use

Grep imports for the supported instrumentations. The result drives the `instrumentations` array.

| If you see in code | Add to `instrumentations` |
| --- | --- |
| `import OpenAI from "openai"` / `from openai import OpenAI` | `"openai"` (also covers `AzureOpenAI` from the same `openai` package) |
| `@anthropic-ai/sdk` / `anthropic` | `"anthropic"` |
| `@aws-sdk/client-bedrock-runtime` (TS) / `boto3` Bedrock client (Py) | `"bedrock"` |
| `cohere-ai` (TS) / `cohere` (Py) | `"cohere"` |
| `together-ai` (TS) / `together` (Py) | `"togetherai"` |
| `@google-cloud/vertexai` (TS) / `google-cloud-aiplatform` (Py) | `"vertexai"` |
| `@google-cloud/aiplatform` (TS) / `google-cloud-aiplatform` (Py) | `"aiplatform"` |
| `langchain`, `@langchain/*` (TS) / `langchain-core` (Py) | `"langchain"` |
| `llamaindex` (TS) / `llama-index` (Py) | `"llamaindex"` |
| `ai` (Vercel AI SDK) | **none** — see special case below |
| `@mastra/core` (Mastra) | **none** — TypeScript-only special case; do not install `@latitude-data/telemetry` at all. See special case below |
| `@openai/agents` (OpenAI Agents SDK) | `"openai"` — Agents SDK calls into the `openai` client under the hood; the patch lands at the chat-completions layer |

Special cases:

- **Vercel AI SDK (`ai`, `@ai-sdk/openai`, etc.)**: do **not** add an instrumentation entry for it. The AI SDK has native OpenTelemetry support. Initialize Latitude (`initLatitude({ apiKey, projectSlug })`) and pass `experimental_telemetry: { isEnabled: true, metadata: {...} }` on each `generateText` / `streamText` call. Latitude's smart filter picks up the AI SDK's `ai.*` spans automatically. Adding `"openai"` here would not produce extra traces and may cause double-counting.
- **Mastra (`@mastra/core`)**: TypeScript only. Mastra ships its own OpenTelemetry pipeline via `@mastra/observability` and `@mastra/otel-exporter`, emitting `gen_ai.*` spans natively. **Do not install `@latitude-data/telemetry`** — the integration is configured entirely through Mastra's `OtelExporter` with a `custom` provider pointed at Latitude's OTLP endpoint and the standard `Authorization` / `X-Latitude-Project` headers. See [references/typescript.md](references/typescript.md#mastra-example-shape) for the full setup. Source: `docs/telemetry/frameworks/mastra.mdx`.
- **LangChain / LlamaIndex**: register the wrapper instrumentation (`"langchain"` / `"llamaindex"`); you do **not** also need to register the underlying vendor.
- **Gemini consumer SDK (`@google/generative-ai`)**: not in the supported list. If the app is on Gemini, ask the user whether they can switch to `@google-cloud/aiplatform` / `@google-cloud/vertexai`, or fall back to manual span creation.
- **Custom HTTP clients (raw `fetch` to OpenAI, Anthropic, etc.)**: not covered by any auto-instrumentation. Either switch to the vendor SDK or write manual spans — `capture()` alone produces no traces.

#### 2e. Where LLM calls happen

This is the step the failed demo skipped. Find the actual call sites:

```bash
# TypeScript / Node
grep -rE "chat\.completions\.create|messages\.create|generateText|streamText|embeddings\.create" .
# Python
grep -rE "chat\.completions\.create|messages\.create|generate_content|complete\(" .
```

For each call site, walk back up the call stack to find the **entry point** (HTTP route handler, queue consumer, CLI command, agent loop iteration). The entry point is where `capture()` belongs — if `capture()` is used at all. Telemetry initialization belongs **earlier**, at process startup.

If you cannot determine the entry point, ask the user: *"I see LLM calls in `X.ts:42`. What triggers this code — an HTTP request, a background job, a CLI command? I need this to know where to wrap the request boundary."*

### Step 3 — Decide the install path

```
Existing OTel TracerProvider in the codebase?
├─ Yes → Advanced path: LatitudeSpanProcessor on existing provider
│        + registerLatitudeInstrumentations(tracerProvider: existing)
└─ No  → Bootstrap path: initLatitude / init_latitude
```

Conflicting LLM-observability vendor?

- **Traceloop** → advanced path, one shared provider, register instrumentations once.
- **Langfuse / LangSmith** → confirm intent with user; if keeping both, advanced path.
- **None** → bootstrap path is fine.

### Step 4 — Place the initialization correctly

Initialization must run **once per process** at startup, **before** the first LLM call. Patch-based auto-instrumentation also requires init to run **before** the LLM client is constructed when the patch hooks the constructor.

#### TypeScript: keep it inline, do not create new files

For TypeScript installs, **do not invent a new telemetry module** (`telemetry.ts`, `lib/latitude.ts`, `setup/observability.ts`, etc.). Add `initLatitude` directly to the file where the LLM call already lives, at the top, before the LLM SDK is used. Less indirection means less to debug when traces don't appear.

Minimal shape — copy this when adding to a single-file script that uses the Vercel AI SDK:

```typescript
import { initLatitude, capture } from "@latitude-data/telemetry";
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

const latitude = initLatitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
});

await latitude.ready; // REQUIRED — see Rule 2 at top of skill

await capture("generate-support-reply", async () => {
  const { text } = await generateText({
    model: openai("gpt-4o"),
    prompt: "Hello",
    experimental_telemetry: { isEnabled: true },
  });
  return text;
});

await latitude.shutdown();
```

The only times you should put init in a separate file are framework-mandated bootstrap files that **already exist** (e.g. Next.js `instrumentation.ts`, NestJS `main.ts`, an existing `server.ts` / `index.ts`). Even then: edit the existing file, do not create a wrapper module around `initLatitude`.

Common locations by stack:

| Stack | Place init in |
| --- | --- |
| Next.js (App Router) | `instrumentation.ts` `register()` hook (Node runtime only) |
| Plain Node server | top of `server.ts` / `index.ts`, before any LLM client import |
| NestJS | `main.ts` before `NestFactory.create` |
| Express / Fastify / Hono | top of the bootstrap file, before route registration |
| Python FastAPI | app factory or `lifespan` startup, before route imports trigger LLM client creation |
| Python Celery | worker `__init__` or `worker_process_init` signal |
| CLI / one-shot script | very top of the entrypoint, before any LLM client import |

For TypeScript, **import order matters**. Telemetry imports first, then the LLM SDK. If the bundler reshuffles imports (Next.js, esbuild with certain settings), use the advanced path with explicit `modules` (see [references/typescript.md](references/typescript.md)).

#### Short-lived and serverless processes

Long-running servers can rely on the batch span processor flushing on its own schedule. Anything that exits quickly cannot:

- CLI scripts, one-shot jobs, notebooks → `await latitude.shutdown()` (TS) or `latitude.shutdown()` (Py) before exit.
- Serverless handlers (Lambda, Vercel Functions, Cloud Run, Cloud Functions) → `await latitude.flush()` after the LLM work completes, before the response is returned (or before the isolate suspends). Decide per route whether the latency cost is acceptable; for streaming responses, flushing after the stream completes is usually correct.
- Next.js App Router on Edge runtime → not supported; force `runtime = "nodejs"` on any route that calls LLM SDKs.

### Step 5 — Add `capture()` only where it makes sense

Re-read the rule at the top of this skill before adding `capture()`. Then:

- Wrap **coarse** boundaries: a request handler, a queue job, an agent turn.
- Wrap code that **definitely** invokes an auto-instrumented LLM SDK.
- Do not wrap pure functions, prompt builders, parsers, validators — they create no spans, so `capture()` decorates nothing.
- Do not wrap the application bootstrap or `register()` hook.

If the user only needs raw spans (no user/session attribution), skip `capture()` entirely. Auto-instrumentation alone produces useful traces.

### Step 6 — Verify before reporting done

Do not declare success based on "the code compiles." Verify a trace lands in Latitude:

1. Run the app (or the relevant script) and trigger one LLM call.
2. Open the Latitude UI → Traces and confirm at least one new trace appears.
3. If no trace:
   - Re-check `LATITUDE_API_KEY` and `LATITUDE_PROJECT_SLUG`.
   - Re-check that the `instrumentations` array includes the SDK actually being called.
   - For TypeScript: try the advanced path with explicit `modules: { openai: OpenAI }` (or matching vendor) — bundler-resolved module instances may not match the SDK's auto-`require`.
   - For short-lived processes: ensure `await latitude.shutdown()` (or `latitude.flush()`) runs before exit.
   - Check whether the smart filter is dropping spans (`disableSmartFilter` for diagnostic runs).

### Step 7 — Add context (optional, infer first, ask second)

Once raw traces are flowing, suggest context additions based on what the codebase already exposes. **Infer from code; only ask when the answer is not in the code.**

| If you see in code | Suggest |
| --- | --- |
| `req.user.id`, `session.userId`, auth middleware | `userId` on the wrapping `capture()` |
| Conversation history, `messages` array, chat threads | `sessionId` |
| Multiple distinct routes / agents | `tags: ["<feature-name>"]` |
| Tenant or workspace identifiers | `metadata: { tenantId }` |
| Environment differentiation (`process.env.NODE_ENV`) | `tags: ["production" | "staging"]` |

Do not blanket-add context. Each addition has a maintenance cost; prefer small, intentional sets.

## Documentation workflow

When package APIs may have drifted, fetch the current README:

- TypeScript: `https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/typescript/README.md`
- Python: `https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/python/README.md`
- Product docs: `https://docs.latitude.so/telemetry/overview`

If the README and this skill disagree, **the README wins**. Offer to update this skill when drift is found.

## When to read which reference

| Situation | Open |
| --- | --- |
| TypeScript / Node specifics, ESM gotchas, `modules` option, per-SDK notes, Next.js | [references/typescript.md](references/typescript.md) |
| Python specifics | [references/python.md](references/python.md) |
| Non-TS/Python codebases (Go, Java, Ruby, .NET, PHP, Rust, Elixir, …) | [references/otlp-fallback.md](references/otlp-fallback.md) |
| Auditing an existing integration or a PR | [references/audit-checklist.md](references/audit-checklist.md) |

## Packages

**Always install the alpha tag.** The stable releases are behind the current SDK surface this skill targets — installing without the alpha tag will pull an older version that is missing APIs referenced here. Do not drop the `@alpha` / `--pre` flag, even if the user does not mention pre-release versions.

| Language | Install | Public surface |
| --- | --- | --- |
| TypeScript / Node | `npm install @latitude-data/telemetry@alpha` | `initLatitude`, `LatitudeSpanProcessor`, `registerLatitudeInstrumentations`, `capture` |
| Python (3.11+) | `pip install --pre latitude-telemetry` | `init_latitude`, `LatitudeSpanProcessor`, `register_latitude_instrumentations`, `capture` |

If the user's package manager is not npm/pip, translate the same intent: `pnpm add @latitude-data/telemetry@alpha`, `yarn add @latitude-data/telemetry@alpha`, `bun add @latitude-data/telemetry@alpha`, `uv pip install --prerelease=allow latitude-telemetry`, `poetry add latitude-telemetry@^0.0.0-alpha` (or `--allow-prereleases`). The alpha channel is mandatory; stable is not yet aligned with this skill.

## Common mistakes

| Mistake | Why it fails | Fix |
| --- | --- | --- |
| `capture()` wraps a non-LLM function | No span is created — `capture()` only decorates existing spans | Move `capture()` to the boundary that calls the LLM SDK |
| `await latitude.ready` skipped | First LLM call races the patch registration; trace silently dropped | Always put `await latitude.ready` directly after `initLatitude` |
| New `telemetry.ts` / `lib/latitude.ts` module created to hold init | Adds an import layer that often runs the LLM SDK before init; pure indirection | Inline `initLatitude` in the file that already runs the LLM call |
| `initLatitude` runs after the LLM client is constructed | Patch never applied | Import telemetry first; in Next.js use `instrumentation.ts` |
| `instrumentations: []` while OpenAI is in use | No vendor patched, no spans | Add `"openai"` (or matching vendor) |
| Adding `"openai"` to `instrumentations` for Vercel AI SDK code | Double-counted spans, confusing traces | The AI SDK has native OTel; just enable `experimental_telemetry: { isEnabled: true }` per call. Do not register `"openai"` for AI SDK paths. |
| Python: `latitude.shutdown()` (attribute access) | `AttributeError` — `init_latitude` returns a dict | Use item access: `latitude["shutdown"]()` and `latitude["flush"]()` |
| Hardcoded API key | Leaks in repo, breaks per-env config | Read from `process.env` / `os.environ` |
| Script exits before flush | Buffered batches dropped | `await latitude.shutdown()` (TS) or `latitude.shutdown()` (Py) |
| Two OTel `TracerProvider` instances | Spans split across providers | Advanced path with one shared provider |
| Edge runtime on Next.js | OTel exporters/patches assume Node | Force `runtime = "nodejs"` on the route |
| Bundler resolves a different `openai` module | Auto-require patches the wrong instance | Pass explicit `modules: { openai: OpenAI }` to `registerLatitudeInstrumentations` |

## Skill maintenance

If this skill gives wrong or outdated guidance, the README under `packages/telemetry/{typescript,python}` and the docs at `docs.latitude.so/telemetry/*` are the source of truth. Offer to file an issue or open a PR against `latitude-dev/skills` when drift is found.
