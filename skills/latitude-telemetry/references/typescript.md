# TypeScript: Latitude Telemetry

Concise reference for `@latitude-data/telemetry`. Always confirm details against the [upstream TypeScript README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/typescript/README.md).

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
  instrumentations: ["openai"], // extend per README
});

// Optional: await latitude.ready before first LLM calls if you must guarantee patches are applied
await latitude.ready;

// ... LLM work ...

await latitude.shutdown();
```

Notes from upstream:

- `initLatitude` returns immediately; instrumentations register in the background.
- Use `await latitude.ready` when you need registration finished before the first LLM call (helps in tests and tight races).

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
});
```

Keep smart filtering and redaction behavior in mind (defaults are documented upstream).

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
| No spans in Latitude | API key / project slug; instrumentations registered; smart filter not hiding non-LLM spans |
| Missing spans at process exit | `flush()` / `shutdown()` |
| `capture()` seems empty | Instrumentation must create child spans; `capture()` only adds attributes |

For Datadog and Sentry composition, copy the vendor sections from the upstream README rather than duplicating them here.
