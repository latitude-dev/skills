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

## Common pitfalls

| Symptom | Things to verify |
| ------- | ---------------- |
| No spans in Latitude | API key / project slug; instrumentations registered; for TS, try explicit `modules` if auto-require fails; smart filter not hiding non-LLM spans |
| Missing spans at process exit | `flush()` / `shutdown()` |
| `capture()` seems empty | Instrumentation must create child spans; `capture()` only adds attributes |

For Datadog and Sentry composition, copy the vendor sections from the upstream README rather than duplicating them here.
