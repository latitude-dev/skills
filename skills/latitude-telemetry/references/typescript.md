# TypeScript: Latitude Telemetry

Concise reference for `@latitude-data/telemetry`. Always confirm details against the [upstream TypeScript README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/typescript/README.md).

The product docs at [Latitude — Developers overview](https://docs.latitude.so/developers/overview) describe the same mental model (install → bootstrap telemetry → wrap work in `capture()`), but API names may differ from this package (for example `LatitudeTelemetry` vs `initLatitude`, or `projectId` / `path` vs `projectSlug` and `capture(name, …)`).

## Install

```bash
npm install @latitude-data/telemetry
```

## Path A — Bootstrap (recommended)

Single entrypoint builds OpenTelemetry, LLM auto-instrumentation, and Latitude export.

```typescript
import { initLatitude } from "@latitude-data/telemetry";

const latitude = initLatitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
  instrumentations: ["openai"], // which providers to enable — see “Instrumentations vs vendor modules” below
});

// Optional: await latitude.ready before first LLM calls if you must guarantee patches are applied
await latitude.ready;

// ... LLM work ...

await latitude.shutdown();
```

Notes from upstream:

- `initLatitude` returns immediately; instrumentations register in the background.
- Use `await latitude.ready` when you need registration finished before the first LLM call (helps in tests and tight races).

### Instrumentations vs vendor modules

`instrumentations` is an array of **string identifiers** (`"openai"`, `"anthropic"`, …). That list tells Latitude **which** Traceloop instrumentations to register. By default the SDK tries to load each vendor package itself (`require` / dynamic `import`).

Patch-based instrumentation only works if the SDK resolves the **same module instance** your app uses. In ESM-first apps, monorepos, or bundled servers, auto-resolution can fail silently. In those cases, use **Path B** and pass explicit **`modules`** (vendor exports) into `registerLatitudeInstrumentations` — for example OpenAI expects the client class you import from `openai`, not only the string `"openai"`:

```typescript
import OpenAI from "openai";
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import {
  LatitudeSpanProcessor,
  registerLatitudeInstrumentations,
} from "@latitude-data/telemetry";

const provider = new NodeTracerProvider({
  spanProcessors: [
    new LatitudeSpanProcessor(
      process.env.LATITUDE_API_KEY!,
      process.env.LATITUDE_PROJECT_SLUG!,
    ),
  ],
});
provider.register();

await registerLatitudeInstrumentations({
  instrumentations: ["openai"],
  modules: { openai: OpenAI },
  tracerProvider: provider,
});
```

`initLatitude` currently forwards only `instrumentations` to the registrar (no `modules` option on `InitLatitudeOptions` in upstream `types.ts`). If you need explicit `modules` with bootstrap-level convenience, either use Path B as above or watch the package README for a future `modules` / `instrumentationModules` option on `initLatitude`.

## Path B — Existing OpenTelemetry (advanced)

Use when a `NodeSDK` / `NodeTracerProvider` already exists or when sending spans to multiple backends.

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import {
  LatitudeSpanProcessor,
  registerLatitudeInstrumentations,
} from "@latitude-data/telemetry";

const sdk = new NodeSDK({
  spanProcessors: [
    /* existing processors */
    new LatitudeSpanProcessor(
      process.env.LATITUDE_API_KEY!,
      process.env.LATITUDE_PROJECT_SLUG!,
    ),
  ],
});

sdk.start();

await registerLatitudeInstrumentations({
  instrumentations: ["openai"],
  tracerProvider: sdk.getTracerProvider(),
  // When auto-require misses the vendor package, pass the same import your app uses:
  // modules: { openai: OpenAI },
});
```

Keep smart filtering and redaction behavior in mind (defaults are documented upstream). Option names in the published README may lag the source; the implementation lives in `packages/telemetry/typescript/src/sdk/instrumentations.ts` (`modules` today).

## Optional context: `capture()`

Wrap request- or agent-level work to stamp `user.id`, `session.id`, `latitude.tags`, and `latitude.metadata` onto spans created inside the callback.

```typescript
import { initLatitude, capture } from "@latitude-data/telemetry";

const latitude = initLatitude({ /* ... */ });

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
  },
);

await latitude.shutdown();
```

## Public surface (import map)

Typical imports:

```typescript
import {
  initLatitude,
  LatitudeSpanProcessor,
  capture,
  registerLatitudeInstrumentations,
} from "@latitude-data/telemetry";
```

## Per-SDK notes

These are the real-world gotchas per LLM SDK. The string identifier alone is often not enough.

The full set of supported identifiers (from `InstrumentationType` in `instrumentations.ts`):

```
"openai" | "anthropic" | "bedrock" | "cohere"
"langchain" | "llamaindex" | "togetherai" | "vertexai" | "aiplatform"
```

| SDK | Notes |
| --- | --- |
| `openai` | If the bundler resolves a different `openai` module than the SDK auto-`require`s, no patch lands. Use Path B with `modules: { openai: OpenAI }`. Same identifier covers `AzureOpenAI` (it's exported from the same `openai` package). |
| `@anthropic-ai/sdk` | Identifier `"anthropic"`. Same `modules` workaround applies if auto-resolution misses. |
| `@aws-sdk/client-bedrock-runtime` | Identifier `"bedrock"`. AWS SDK v3 uses ESM exports; verify spans actually appear before declaring done. |
| `cohere-ai`, `together-ai`, `@google-cloud/vertexai`, `@google-cloud/aiplatform` | Identifiers `"cohere"`, `"togetherai"`, `"vertexai"`, `"aiplatform"` respectively. |
| `langchain` / `@langchain/*` | Identifier `"langchain"`. Wraps LangChain's internal callbacks; you do **not** also need to register the underlying vendor (e.g. `"openai"`) when LangChain is the only path. |
| `llamaindex` | Identifier `"llamaindex"`. Same wrapper-level instrumentation as LangChain. |
| **Vercel AI SDK (`ai`, `@ai-sdk/openai`, …)** | **No instrumentations identifier.** The AI SDK ships native OTel support. Initialize Latitude without listing it: `initLatitude({ apiKey, projectSlug })`. Then on each AI SDK call, set `experimental_telemetry: { isEnabled: true, metadata: { ... } }`. Latitude's smart filter picks up the SDK's `ai.*` spans automatically. Do not also register `"openai"` — it would double-count. |
| **OpenAI Agents SDK (`@openai/agents`)** | No dedicated identifier. The Agents SDK calls into the `openai` client under the hood; register `"openai"` and the patch lands at the chat-completions layer, so each agent step is captured. |
| **Gemini consumer SDK (`@google/generative-ai`)** | Not in the supported list. The `"aiplatform"` identifier patches `@google-cloud/aiplatform`, which is a different package. If the app is on Gemini, prefer migrating to `@google-cloud/aiplatform` or write manual spans. |
| Custom HTTP clients (raw `fetch` to OpenAI, etc.) | Not covered by any auto-instrumentation. Either switch to the vendor SDK or write manual spans — `capture()` alone will not produce traces. |

If a wrapper library is the only path used, register only the wrapper's instrumentation, not the vendor under it. If application code mixes both, register both.

### Vercel AI SDK example shape

```typescript
import { initLatitude, capture } from "@latitude-data/telemetry";
import { openai } from "@ai-sdk/openai";
import { generateText } from "ai";

const latitude = initLatitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  projectSlug: process.env.LATITUDE_PROJECT_SLUG!,
  // No instrumentations array — the AI SDK provides its own OTel spans.
});

await latitude.ready;

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

## Next.js notes

Two things are genuinely Next.js-specific. Everything else is generic Node guidance from the sections above.

- **`instrumentation.ts` is the right place.** Next.js calls `register()` on server startup before route modules load. Put `initLatitude` (or the advanced-path setup) inside `register()` so every Route Handler and Server Action shares the same provider. If `instrumentation.ts` is not an option (legacy app, incremental adoption), put init in a shared `server-only` module imported first by every server entry point.
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
