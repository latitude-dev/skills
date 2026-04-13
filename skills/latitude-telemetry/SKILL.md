---
name: latitude-telemetry
description: Adds or audits Latitude LLM observability using OpenTelemetry-based Latitude Telemetry (TypeScript `@latitude-data/telemetry`, Python `latitude-telemetry`). Covers bootstrap setup (`initLatitude` / `init_latitude`), existing OpenTelemetry integration (`LatitudeSpanProcessor`, `registerLatitudeInstrumentations` / Python equivalents), optional `capture()` for user/session/tags/metadata, env vars, troubleshooting, and Vercel / Next.js (Node runtime) patterns. Use when the user mentions Latitude, latitude.so, Latitude telemetry, LLM tracing to Latitude, or OpenTelemetry export to Latitude alongside other backends.
---

# Latitude Telemetry

Help the user ship reliable LLM traces to [Latitude](https://latitude.so) using the official telemetry packages in [latitude-llm/packages/telemetry](https://github.com/latitude-dev/latitude-llm/tree/main/packages/telemetry).

## Core principles

1. **Upstream first**: Before editing code, fetch the current package README from the repo (TypeScript and Python READMEs change more often than this skill). Prefer the raw README URLs under `packages/telemetry/{typescript,python}/README.md` on the default branch.
2. **Two setup paths**: Default is **bootstrap** (`initLatitude` / `init_latitude`). Use **existing OpenTelemetry** when the app already has a `TracerProvider` / `NodeSDK` or another vendor (Datadog, Sentry, Jaeger).
3. **Instrumentation creates spans; `capture()` adds context**: `capture()` does not create spans. It attaches Latitude context to spans created by LLM auto-instrumentation.
4. **Order matters (TypeScript)**: Register Latitude and instrumentations before creating LLM clients when patching-based auto-instrumentation is involved—mirror the “import order” lessons from other LLM SDK wrappers.
5. **TypeScript `instrumentations` is not “just a string” for patching**: the `instrumentations` array selects which providers to register; when auto-loading the vendor package fails (bundlers, ESM, monorepos), pass the real SDK export via `modules` on `registerLatitudeInstrumentations` (see [references/typescript.md](references/typescript.md)). This matches the intent described in [Latitude developers overview](https://docs.latitude.so/developers/overview) even though V1 docs may show a single object map on a different constructor.
6. **Flush on short-lived processes**: Scripts, tests, and some serverless handlers must `flush()` / `shutdown()` (or `forceFlush()` on the provider) so batches reach Latitude before the process exits.

## When to read which reference

| Situation | Open |
| --------- | ---- |
| TypeScript / Node (including manual vs advanced OTel) | [references/typescript.md](references/typescript.md) |
| Python | [references/python.md](references/python.md) |
| Next.js on Vercel, App Router, Route Handlers, `instrumentation.ts` | [references/vercel-nextjs.md](references/vercel-nextjs.md) |
| Checklist: auditing an integration | [references/audit-checklist.md](references/audit-checklist.md) |

## Credentials and environment

| Variable | Role |
| -------- | ---- |
| `LATITUDE_API_KEY` | Auth for ingest |
| `LATITUDE_PROJECT_SLUG` | Target project in Latitude |
| `LATITUDE_TELEMETRY_URL` | Optional OTLP endpoint override (see upstream README defaults for dev vs prod) |

On Vercel, define these in Project Settings → Environment Variables for Production / Preview / Development as appropriate.

## Documentation workflow

1. Fetch the TypeScript README: `https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/typescript/README.md`
2. Fetch the Python README: `https://raw.githubusercontent.com/latitude-dev/latitude-llm/main/packages/telemetry/python/README.md`
3. Implement using the fetched content as the source of truth for API shapes, option names, and vendor-specific examples (Datadog, Sentry, and so on).

## Quick decision tree

```text
Greenfield app?
├─ Yes → Bootstrap: initLatitude / init_latitude + chosen instrumentations
└─ No → Already using OpenTelemetry?
         ├─ Yes → Add LatitudeSpanProcessor (+ register* instrumentations on that provider)
         └─ No  → Bootstrap unless user explicitly needs a custom provider layout
```

## Packages

| Language | Install | Notes |
| -------- | ------- | ----- |
| TypeScript / Node | `npm install @latitude-data/telemetry` | See TS README for `initLatitude`, `LatitudeSpanProcessor`, `registerLatitudeInstrumentations`, `capture` |
| Python | `pip install latitude-telemetry` | Python 3.11+; see Py README for `init_latitude`, `LatitudeSpanProcessor`, `register_latitude_instrumentations`, `capture` |

Supported LLM instrumentation identifiers and matching client libraries are listed in each README (OpenAI, Anthropic, Bedrock, LangChain, and others). In TypeScript, you can still need explicit **`modules`** entries when string-only registration does not resolve the same vendor package your app uses.

## Skill maintenance

If instructions here disagree with the fetched README, **the README wins**. Offer to update this skill when the user reports drift.
