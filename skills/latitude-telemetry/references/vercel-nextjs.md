# Vercel and Next.js

Latitude’s Node packages rely on OpenTelemetry patterns that assume a **Node.js** runtime. Treat this reference as deployment guidance; still fetch the [TypeScript package README](https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/typescript/README.md) for API details.

## Runtime selection

- Prefer **`runtime = "nodejs"`** (explicitly or by default) for Route Handlers, Server Actions, and background jobs that call LLM clients instrumented by `@latitude-data/telemetry`.
- **Avoid the Edge runtime** for code paths that initialize OpenTelemetry exporters and patch-based LLM instrumentations unless you have verified upstream support for that environment.

## Where to initialize

Goal: run `initLatitude` **once per Node server / isolate lifecycle**, as early as practical, before LLM clients are imported or constructed when patch order matters.

### `instrumentation.ts` (App Router)

Next.js supports a root `instrumentation.ts` (or `src/instrumentation.ts`) `register` hook for server-side setup.

- Start Latitude telemetry in `register()` when the process boots so subsequent Route Handlers share the same provider.
- Do not move heavy synchronous blocking work into middleware if it slows every request; telemetry init should follow upstream guidance (the TS bootstrap is designed to return quickly while instrumentations register in the background).

### Route Handlers and Server Actions

If you cannot use `instrumentation.ts` (legacy layout or incremental adoption), initialize in a **shared server-only module** imported first by every LLM entrypoint, or at the top of a dedicated `lib/latitude.ts` that every server route imports before OpenAI / Anthropic clients.

## Serverless and flushing

Vercel Functions can freeze or terminate shortly after the response is sent.

- After important LLM work in short handlers, call **`await latitude.flush()`** (or `shutdown()` when tearing down a script) when you must guarantee delivery before the isolate suspends.
- For streaming responses, decide whether to flush after the stream completes (trade latency vs trace completeness).

## Environment variables

Configure in the Vercel dashboard:

- `LATITUDE_API_KEY`
- `LATITUDE_PROJECT_SLUG`
- `LATITUDE_TELEMETRY_URL` when pointing at self-hosted ingest or non-default endpoints

Load them on the server only; never expose secrets to the browser bundle.

## Monorepos and Turborepo

Install `@latitude-data/telemetry` in the workspace package that executes LLM calls (for example `apps/web` or `packages/ai`). Keep a single initialization path per deployable Node service.

## Vercel AI SDK and other frameworks

If the stack uses the Vercel AI SDK or wrappers, ensure the **underlying vendor client** matches a supported `instrumentations` entry from the Latitude README. If spans are missing, verify import order and, on TypeScript, pass the **same** vendor module via `modules` on `registerLatitudeInstrumentations` when Next’s bundler does not align with auto-`require` (see [typescript.md](typescript.md)).
