# TypeScript: Latitude Telemetry

Concise reference for `@latitude-data/telemetry`. Always confirm details against the [upstream TypeScript README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/typescript/README.md).

The product docs at [Latitude — Developers overview](https://docs.latitude.so/developers/overview) describe the same mental model (install → bootstrap telemetry → wrap work in `capture()`), but API names may differ from this package (for example `LatitudeTelemetry` vs `initLatitude`, or `projectId` / `path` vs `projectSlug` and `capture(name, …)`).

## Install

**Always look up the current pre-release and pin to it.** The SDK is on the `alpha` / `beta` channel and the API surface changes between releases. The `@alpha` dist-tag floats, so a CI re-run weeks later may land on a different version with a different API surface. Look up once, pin exact.

```bash
# Find the current version (e.g. 3.0.0-alpha.10).
npm view @latitude-data/telemetry@alpha version
# If the SDK has moved to beta, this returns nothing — try @beta:
npm view @latitude-data/telemetry@beta version

# Pin to the exact version returned.
npm install @latitude-data/telemetry@3.0.0-alpha.10
# Or: pnpm add / yarn add / bun add with the same exact-version syntax.
```

Confirm the lockfile (`package-lock.json` / `pnpm-lock.yaml` / `yarn.lock`) captures the pin. Without a lock, the install will drift again.

## Path A — Bootstrap (recommended)

`new Latitude({...})` is the canonical entry point. It auto-detects an existing OpenTelemetry `TracerProvider` (registered globally or passed via `tracerProvider`) and attaches its span processor to it; if none exists, it creates and registers one. Path B (manual `LatitudeSpanProcessor` + `registerLatitudeInstrumentations`) is reserved for cases where you already build the provider yourself and want full control.

**Keep it inline — do NOT create a dedicated `telemetry.ts` / `lib/latitude.ts` module just to hold this.** Put the four lines below at the top of the file that already runs the LLM call. Wrapping the bootstrap in a helper module is a frequent source of import-order bugs (the helper imports the LLM SDK before init runs) and adds nothing.

```typescript
import { Latitude, capture } from "@latitude-data/telemetry";
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
});

await latitude.ready; // REQUIRED — never skip this

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

When using a non-AI-SDK vendor (raw `openai`, `@anthropic-ai/sdk`, …), add the matching identifier to `instrumentations`, e.g. `instrumentations: ["openai"]`. See "Instrumentations vs vendor modules" below.

`await latitude.ready` is **required, not optional.** `new Latitude({...})` returns immediately and patches run in the background; without `await latitude.ready`, the first LLM call can fire before the patch lands and the trace is silently lost. Past installs by this skill have shipped without it and produced empty trace lists. If you find yourself tempted to skip it, stop — there is no scenario in this skill where omitting it is correct.

### `projectSlug` is optional on the constructor

If your app emits to several Latitude projects (multi-agent, multi-feature), omit `projectSlug` from the constructor and pass it on each `capture()` call instead. See [project-scoping.md](project-scoping.md) for the full pattern and precedence rules. For single-project apps, keep `projectSlug` in the constructor — it's the simpler shape.

### Legacy `initLatitude`

`initLatitude({...})` still exists as a thin wrapper around `new Latitude({...})` for backwards compatibility with older code samples. New installs should use the class directly. If you're reading an existing codebase that calls `initLatitude`, leave it alone — the wrapper is supported — but suggest the class form when adding new init.

### Instrumentations vs vendor modules

`instrumentations` is an array of **string identifiers** (`"openai"`, `"anthropic"`, …). That list tells Latitude **which** Traceloop instrumentations to register. By default the SDK tries to load each vendor package itself (`require` / dynamic `import`).

Patch-based instrumentation only works if the SDK resolves the **same module instance** your app uses. In ESM-first apps, monorepos, or bundled servers, auto-resolution can fail silently. In those cases, drop down to `registerLatitudeInstrumentations` directly and pass explicit **`modules`** (vendor exports) — for example OpenAI expects the client class you import from `openai`, not only the string `"openai"`:

```typescript
import OpenAI from "openai";
import { Latitude, registerLatitudeInstrumentations } from "@latitude-data/telemetry";

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
  // Skip `instrumentations` here — we'll register manually below with explicit modules.
});

await registerLatitudeInstrumentations({
  instrumentations: ["openai"],
  modules: { openai: OpenAI },
  tracerProvider: latitude.provider,
});

await latitude.ready; // Still required for any instrumentations passed to the constructor.
```

The constructor's `instrumentations` option does not currently accept `modules`; if you need explicit module overrides, register manually as above. Watch the package README for a future `instrumentationModules` field on `LatitudeOptions`.

## Path B — Existing OpenTelemetry (advanced)

If your app already builds a `NodeSDK` / `NodeTracerProvider` and you want to keep full control of it, hand it to the constructor:

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { Latitude } from "@latitude-data/telemetry";

const sdk = new NodeSDK({
  spanProcessors: [/* existing processors */],
});
sdk.start();

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
  instrumentations: ["openai"],
  tracerProvider: sdk.getTracerProvider(),
});

await latitude.ready;
```

The constructor attaches `LatitudeSpanProcessor` to the provider you passed; your existing processors continue to receive every span. If you skip `tracerProvider`, the SDK auto-detects a globally-registered provider — same outcome with fewer lines.

For the rare case where you want to manage everything yourself (no constructor at all), the lower-level primitives are still exported:

```typescript
import { LatitudeSpanProcessor, registerLatitudeInstrumentations } from "@latitude-data/telemetry";

provider.addSpanProcessor(new LatitudeSpanProcessor(apiKey, projectSlug));

await registerLatitudeInstrumentations({
  instrumentations: ["openai"],
  modules: { openai: OpenAI },
  tracerProvider: provider,
});
```

Use this only when the constructor's auto-detection genuinely doesn't fit (multi-process / multi-provider exotic setups). For everything else, `new Latitude({...})` is shorter and less error-prone. Option names in the published README may lag the source; the implementation lives in `packages/telemetry/typescript/src/sdk/instrumentations.ts` (`modules` today).

## Optional context: `capture()`

Wrap request- or agent-level work to stamp `user.id`, `session.id`, `latitude.tags`, `latitude.metadata`, and (optionally) `latitude.project` onto spans created inside the callback.

```typescript
import { Latitude, capture } from "@latitude-data/telemetry";

const latitude = new Latitude({ /* ... */ });
await latitude.ready;

await capture(
  "handle-user-request",
  async () => {
    /* LLM calls */
  },
  {
    userId: "user_123",
    sessionId: "session_abc",
    tags: ["production"],
    metadata: { requestId: "req-xyz" },
    // Optional — routes this capture (and its children) to a different Latitude
    // project than the constructor's default. See project-scoping.md.
    // projectSlug: "evaluation-runs",
  },
);

await latitude.shutdown();
```

For the multi-project pattern (per-capture `projectSlug` override, OTEL resource attribute alternative, bare-OTel routing), see [project-scoping.md](project-scoping.md).

## Public surface (import map)

Typical imports:

```typescript
import {
  Latitude,
  LatitudeSpanProcessor,
  capture,
  registerLatitudeInstrumentations,
} from "@latitude-data/telemetry";
```

`initLatitude` is still exported for backwards compatibility but is just a wrapper around `new Latitude({...})`; prefer the class form for new code.

## Per-SDK notes

These are the real-world gotchas per LLM SDK. The string identifier alone is often not enough.

The full set of supported identifiers (from `InstrumentationType` in `instrumentations.ts`):

```
"openai" | "openai-agents" | "anthropic" | "bedrock" | "cohere"
"langchain" | "llamaindex" | "togetherai" | "vertexai" | "aiplatform"
```

**The pattern is uniform:** every supported framework / SDK is enabled by adding its identifier string to `instrumentations`. There is no special wiring for any of them — pick the right identifier from the list above and pass it in. Do not invent identifiers (e.g. `"openai-agent"` singular is wrong, `"openai-agents"` is correct).

| SDK | Notes |
| --- | --- |
| `openai` | If the bundler resolves a different `openai` module than the SDK auto-`require`s, no patch lands. Use Path B with `modules: { openai: OpenAI }`. Same identifier covers `AzureOpenAI` (it's exported from the same `openai` package). |
| `@anthropic-ai/sdk` | Identifier `"anthropic"`. Same `modules` workaround applies if auto-resolution misses. |
| `@aws-sdk/client-bedrock-runtime` | Identifier `"bedrock"`. AWS SDK v3 uses ESM exports; verify spans actually appear before declaring done. |
| `cohere-ai`, `together-ai`, `@google-cloud/vertexai`, `@google-cloud/aiplatform` | Identifiers `"cohere"`, `"togetherai"`, `"vertexai"`, `"aiplatform"` respectively. |
| `langchain` / `@langchain/*` | Identifier `"langchain"`. Wraps LangChain's internal callbacks; you do **not** also need to register the underlying vendor (e.g. `"openai"`) when LangChain is the only path. |
| `llamaindex` | Identifier `"llamaindex"`. Same wrapper-level instrumentation as LangChain. |
| **Vercel AI SDK (`ai`, `@ai-sdk/openai`, …)** | **No instrumentations identifier.** The AI SDK ships native OTel support. Initialize Latitude without listing it: `initLatitude({ apiKey, projectSlug })`. Then on each AI SDK call, set `experimental_telemetry: { isEnabled: true, metadata: { ... } }`. Latitude's smart filter picks up the SDK's `ai.*` spans automatically. Do not also register `"openai"` — it would double-count. |
| **Mastra (`@mastra/core`)** | **Do not install `@latitude-data/telemetry` at all.** Mastra ships its own OTel pipeline via `@mastra/observability` + `@mastra/otel-exporter`, emitting `gen_ai.*` spans natively. Configure Mastra's `OtelExporter` with a `custom` provider pointed at Latitude's OTLP endpoint. See "Mastra example shape" below. |
| **OpenAI Agents SDK (`@openai/agents`)** | Identifier `"openai-agents"`. This is a dedicated instrumentation — do **not** register `"openai"` for the Agents SDK; that was earlier guidance and it is wrong. Install both packages (substituting the version you got from Step 2): `npm install @latitude-data/telemetry@<version> @openai/agents`. Then `new Latitude({ apiKey, projectSlug, instrumentations: ["openai-agents"] })`. Source: [docs.latitude.so/telemetry/frameworks/openai-agents](https://docs.latitude.so/telemetry/frameworks/openai-agents). |
| **Gemini consumer SDK (`@google/generative-ai`)** | Not in the supported list. The `"aiplatform"` identifier patches `@google-cloud/aiplatform`, which is a different package. If the app is on Gemini, prefer migrating to `@google-cloud/aiplatform` or write manual spans. |
| Custom HTTP clients (raw `fetch` to OpenAI, etc.) | Not covered by any auto-instrumentation. Either switch to the vendor SDK or write manual spans — `capture()` alone will not produce traces. |

If a wrapper library is the only path used, register only the wrapper's instrumentation, not the vendor under it. If application code mixes both, register both.

### Vercel AI SDK example shape

```typescript
import { Latitude, capture } from "@latitude-data/telemetry";
import { openai } from "@ai-sdk/openai";
import { generateText } from "ai";

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
  // No instrumentations array — the AI SDK provides its own OTel spans.
});

await latitude.ready; // REQUIRED — never skip this

await capture(
  "handle-chat",
  async () => {
    const result = await generateText({
      model: openai("gpt-4o-mini"),
      prompt: "...",
      experimental_telemetry: {
        isEnabled: true,
        metadata: { feature: "chat" },
      },
    });
    return result.text;
  },
  { userId: "user_123", tags: ["production"] },
);
```

### Mastra example shape

Mastra is a special case: it has its own OTel SDK and exporter pipeline, so you do **not** install `@latitude-data/telemetry` for a Mastra-only app. Wire the Latitude OTLP endpoint directly into Mastra's `OtelExporter`.

```bash
npm install @mastra/observability @mastra/otel-exporter @opentelemetry/exporter-trace-otlp-proto
```

```typescript
import { Mastra } from "@mastra/core";
import { Agent } from "@mastra/core/agent";
import { Observability } from "@mastra/observability";
import { OtelExporter } from "@mastra/otel-exporter";

const latitudeExporter = new OtelExporter({
  provider: {
    custom: {
      endpoint: "https://ingest.latitude.so/v1/traces",
      protocol: "http/protobuf",
      headers: {
        Authorization: `Bearer ${process.env.LATITUDE_API_KEY!}`,
        "X-Latitude-Project": process.env.LATITUDE_PROJECT_SLUG!,
      },
    },
  },
});

const agent = new Agent({
  id: "my-agent",
  name: "My Agent",
  model: { provider: "OPEN_AI", name: "gpt-4o" },
  instructions: "You are a helpful assistant.",
});

const mastra = new Mastra({
  agents: { "my-agent": agent },
  observability: new Observability({
    configs: {
      otel: {
        serviceName: "my-mastra-app",
        exporters: [latitudeExporter],
      },
    },
  }),
});
```

Notes:

- `protocol: "http/protobuf"` matches Mastra's docs; the `@opentelemetry/exporter-trace-otlp-proto` package must be installed.
- Latitude UI features that depend on `latitude.tags`, `user.id`, `session.id`, etc. require setting those as standard OTel span attributes through Mastra's own context APIs. The `capture()` helper from `@latitude-data/telemetry` will not work here because there is no Latitude `TracerProvider` in the Mastra setup.
- Source of truth: `docs/telemetry/frameworks/mastra.mdx` in the latitude-llm repo.

## Next.js notes

Two things are genuinely Next.js-specific. Everything else is generic Node guidance from the sections above.

- **`instrumentation.ts` is the right place.** Next.js calls `register()` on server startup before route modules load. Put `new Latitude({...})` inside `register()` so every Route Handler and Server Action shares the same provider. If `instrumentation.ts` is not an option (legacy app, incremental adoption), put init in a shared `server-only` module imported first by every server entry point.
- **Do not run instrumented code on the Edge runtime.** OTel exporters and patch-based instrumentations assume Node. Force `runtime = "nodejs"` on any Route Handler or Server Action that calls an LLM SDK.

## Common pitfalls

| Symptom | Things to verify |
| ------- | ---------------- |
| No spans in Latitude | API key / project slug; instrumentations registered; for TS, try explicit `modules` if auto-require fails; smart filter not hiding non-LLM spans |
| Missing spans at process exit | `flush()` / `shutdown()` |
| `capture()` seems empty | Instrumentation must create child spans; `capture()` only adds attributes |
| Spans missing on Next.js | `instrumentation.ts` not wired, or route is on the Edge runtime |
| Bundler resolved a different vendor module | Pass explicit `modules: { openai: OpenAI }` (or matching vendor) to `registerLatitudeInstrumentations` |

For Datadog and Sentry composition, copy the vendor sections from the upstream README rather than duplicating them here.
