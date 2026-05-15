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

When using a non-AI-SDK vendor (raw `openai`, `@anthropic-ai/sdk`, …), pass the LLM SDK module through the `instrumentations` object, e.g. `instrumentations: { openai: OpenAI }`. See "The `instrumentations` object" below.

`await latitude.ready` is **required, not optional.** `new Latitude({...})` returns immediately and patches run in the background; without `await latitude.ready`, the first LLM call can fire before the patch lands and the trace is silently lost. Past installs by this skill have shipped without it and produced empty trace lists. If you find yourself tempted to skip it, stop — there is no scenario in this skill where omitting it is correct.

### `projectSlug` is optional on the constructor

If your app emits to several Latitude projects (multi-agent, multi-feature), omit `projectSlug` from the constructor and pass it on each `capture()` call instead. See [project-scoping.md](project-scoping.md) for the full pattern and precedence rules. For single-project apps, keep `projectSlug` in the constructor — it's the simpler shape.

### Legacy `initLatitude`

`initLatitude({...})` still exists as a thin wrapper around `new Latitude({...})` for backwards compatibility with older code samples. New installs should use the class directly. If you're reading an existing codebase that calls `initLatitude`, leave it alone — the wrapper is supported — but suggest the class form when adding new init.

### The `instrumentations` object

`instrumentations` is a plain object: keys are integration names (`openai`, `anthropic`, `bedrock`, …) and values are the LLM SDK module the consumer's app code imports. Because the module reference comes from the consumer's own `import`, the patch lands on the same prototype their code invokes — sidestepping the CJS/ESM dual-load class of bugs that the older string-array form silently produced.

```typescript
import OpenAI from "openai";
import * as AnthropicSDK from "@anthropic-ai/sdk";
import { Latitude } from "@latitude-data/telemetry";

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
  instrumentations: { openai: OpenAI, anthropic: AnthropicSDK },
});

await latitude.ready;
```

Module-shape rules:

- **`openai`** accepts either the default export (`import OpenAI from "openai"`) or the namespace (`import * as OpenAINS from "openai"`). The SDK unwraps the namespace to the `OpenAI` class internally.
- **`anthropic`** requires the namespace (`import * as AnthropicSDK from "@anthropic-ai/sdk"`). The underlying Traceloop patch reads `module.Anthropic.Messages.prototype` — passing the bare default class also works because the SDK rewraps it, but the namespace form is the recommended shape.
- **All others** (`bedrock`, `cohere`, `langchain`, `llamaindex`, `togetherai`, `vertexai`, `aiplatform`, `openai-agents`) take the package namespace via `import * as X from "<package>"`.

### Legacy `instrumentations: ["openai"]` is removed — and you MUST force-migrate if you see it

The bare-string-array form is removed in `3.0.0-alpha.11`+ with **no fallback**. Any non-object value — including the old string array — **throws at register time**: `[Latitude] instrumentations must be an object mapping integration names to LLM SDK modules (e.g. { openai: OpenAI, anthropic: AnthropicSDK })`.

The form was removed (not deprecated with a warning) because it silently produced "no spans appear" in any consumer where the LLM SDK ships dual CJS/ESM bundles — every modern one does. The SDK's internal `require` resolved the CJS build while the consumer app loaded the ESM build, so the patch landed on a class the consumer never invoked. Keeping a deprecation fallback would have kept that bug reachable.

#### Migration recipe

Any time you touch a codebase that has `@latitude-data/telemetry` installed, run this check first — *before* doing whatever the user asked for. It is not optional.

1. **Read the installed version** (from `package.json` + lockfile). If `< 3.0.0-alpha.11`, you are mid-breaking-upgrade.
2. **Grep for every legacy call site:** `rg -n 'instrumentations:\s*\[' .` — every match must be rewritten.
3. **Rewrite each match** as follows. For each integration, add the matching import at the top of the file if it isn't already there:

   ```diff
   - import { Latitude } from "@latitude-data/telemetry";
   + import OpenAI from "openai";
   + import * as AnthropicSDK from "@anthropic-ai/sdk";
   + import { Latitude } from "@latitude-data/telemetry";

     new Latitude({
       apiKey: process.env.LATITUDE_API_KEY!,
       projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
   -   instrumentations: ["openai", "anthropic"],
   +   instrumentations: { openai: OpenAI, anthropic: AnthropicSDK },
     });
   ```

4. **Drop the `modules:` option** if any `registerLatitudeInstrumentations({ ..., modules: { ... } })` calls exist — that option was removed in `alpha.11`. Pass the SDK module under its integration key on `instrumentations` instead.
5. **Bump the version pin** to the latest alpha returned by `npm view @latitude-data/telemetry@alpha version`, and update the lockfile.
6. **Compile + run once** to confirm the bootstrap doesn't throw the `must be an object mapping…` `TypeError`.

If you only run step 5 (bump the version) and skip steps 1–4 (migrate the call sites), the app's bootstrap throws on the next start. If you only run steps 1–4 (migrate) and skip step 5 (bump), the user's next CI / Renovate bump will trip the same migration with no agent context to help. Always do both, in the same PR.

## Path B — Existing OpenTelemetry (advanced)

If your app already builds a `NodeSDK` / `NodeTracerProvider` and you want to keep full control of it, hand it to the constructor:

```typescript
import OpenAI from "openai";
import { NodeSDK } from "@opentelemetry/sdk-node";
import { Latitude } from "@latitude-data/telemetry";

const sdk = new NodeSDK({
  spanProcessors: [/* existing processors */],
});
sdk.start();

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
  instrumentations: { openai: OpenAI },
  tracerProvider: sdk.getTracerProvider(),
});

await latitude.ready;
```

The constructor attaches `LatitudeSpanProcessor` to the provider you passed; your existing processors continue to receive every span. If you skip `tracerProvider`, the SDK auto-detects a globally-registered provider — same outcome with fewer lines.

For the rare case where you want to manage everything yourself (no constructor at all), the lower-level primitives are still exported:

```typescript
import OpenAI from "openai";
import {
  LatitudeSpanProcessor,
  registerLatitudeInstrumentations,
} from "@latitude-data/telemetry";

provider.addSpanProcessor(new LatitudeSpanProcessor(apiKey, projectSlug));

await registerLatitudeInstrumentations({
  instrumentations: { openai: OpenAI },
  tracerProvider: provider,
});
```

Use this only when the constructor's auto-detection genuinely doesn't fit (multi-process / multi-provider exotic setups). For everything else, `new Latitude({...})` is shorter and less error-prone.

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

These are the real-world gotchas per LLM SDK. Every supported framework / SDK is enabled by setting its key on the `instrumentations` object to the LLM SDK module the consumer imports.

The full set of supported keys:

```
openai, openai-agents, anthropic, bedrock, cohere,
langchain, llamaindex, togetherai, vertexai, aiplatform
```

| SDK | Object entry + module shape |
| --- | --- |
| `openai` | `openai: OpenAI` — accepts the default export or the namespace. Same key covers `AzureOpenAI` (exported from the same `openai` package). |
| `@anthropic-ai/sdk` | `anthropic: AnthropicSDK` — pass `import * as AnthropicSDK from "@anthropic-ai/sdk"`. The patch reads `module.Anthropic.Messages.prototype`; the SDK rewraps a bare class for you but the namespace is the recommended shape. |
| `@aws-sdk/client-bedrock-runtime` | `bedrock: BedrockNS` — pass `import * as BedrockNS from "@aws-sdk/client-bedrock-runtime"`. AWS SDK v3 uses ESM exports; verify spans actually appear before declaring done. |
| `cohere-ai`, `together-ai`, `@google-cloud/vertexai`, `@google-cloud/aiplatform` | `cohere: CohereNS`, `togetherai: TogetherNS`, `vertexai: VertexAINS`, `aiplatform: AIPlatformNS` — each takes the package namespace. |
| `langchain` / `@langchain/*` | `langchain: LangChainNS` — wraps LangChain's internal callbacks; you do **not** also need the underlying vendor key when LangChain is the only path. |
| `llamaindex` | `llamaindex: LlamaIndexNS` — same wrapper-level instrumentation as LangChain. |
| **Vercel AI SDK (`ai`, `@ai-sdk/openai`, …)** | **No `instrumentations` entry.** The AI SDK ships native OTel support. Initialize Latitude without `instrumentations`. Then on each AI SDK call, set `experimental_telemetry: { isEnabled: true, metadata: { ... } }`. Latitude's smart filter picks up the SDK's `ai.*` spans automatically. Do not also set `openai: OpenAI` — it would double-count. |
| **Mastra (`@mastra/core`)** | **Do not install `@latitude-data/telemetry` at all.** Mastra ships its own OTel pipeline via `@mastra/observability` + `@mastra/otel-exporter`, emitting `gen_ai.*` spans natively. Configure Mastra's `OtelExporter` with a `custom` provider pointed at Latitude's OTLP endpoint. See "Mastra example shape" below. |
| **OpenAI Agents SDK (`@openai/agents`)** | `"openai-agents": OpenAIAgents` — pass `import * as OpenAIAgents from "@openai/agents"`. This is a dedicated instrumentation — do **not** set `openai:` for the Agents SDK. Source: [docs.latitude.so/telemetry/frameworks/openai-agents](https://docs.latitude.so/telemetry/frameworks/openai-agents). |
| **Gemini consumer SDK (`@google/generative-ai`)** | Not in the supported list. The `aiplatform` key patches `@google-cloud/aiplatform`, which is a different package. If the app is on Gemini, prefer migrating to `@google-cloud/aiplatform` or write manual spans. |
| Custom HTTP clients (raw `fetch` to OpenAI, etc.) | Not covered by any auto-instrumentation. Either switch to the vendor SDK or write manual spans — `capture()` alone will not produce traces. |

If a wrapper library is the only path used, register only the wrapper's helper, not the vendor under it. If application code mixes both, register both.

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
