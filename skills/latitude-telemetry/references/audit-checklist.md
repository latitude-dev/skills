# Audit checklist: Latitude telemetry

Use when reviewing an existing integration or a pull request that touches observability.

## Bootstrap vs OpenTelemetry

- [ ] **Bootstrap** (`initLatitude` / `init_latitude`) is used for new apps unless multiple span processors are required.
- [ ] **Advanced** path adds `LatitudeSpanProcessor` to an existing provider and calls `registerLatitudeInstrumentations` / `register_latitude_instrumentations` with the **same** `TracerProvider`.

## Correctness

- [ ] `LATITUDE_API_KEY` and `LATITUDE_PROJECT_SLUG` are read from the environment on the server, not hard-coded.
- [ ] LLM client imports occur **after** telemetry bootstrap when patch-based auto-instrumentation requires it.
- [ ] `instrumentations` array includes every vendor SDK actually used (OpenAI, Anthropic, and so on).
- [ ] Short-lived processes call `flush()` / `shutdown()` (or provider `forceFlush()`).

## Context and privacy

- [ ] `capture()` wraps **coarse** boundaries (request, job, agent turn), not every internal helper.
- [ ] Tags and metadata avoid secrets (tokens, raw prompts if policy forbids).
- [ ] Redaction defaults are understood; custom `redact` / `disable_redact` choices are intentional.

## Platform (Vercel / Next.js)

- [ ] LLM code runs on **Node.js** runtime, not Edge, unless explicitly validated.
- [ ] Initialization happens once per process via `instrumentation.ts` or a shared server module.
- [ ] Serverless handlers that must not lose spans await `flush()` where appropriate.

## Observability of the observability

- [ ] If spans are missing, smart filter settings were checked (`disableSmartFilter` only when justified).
- [ ] For multi-backend setups, spans reach both Latitude and the other vendor per README examples (Datadog, Sentry).
