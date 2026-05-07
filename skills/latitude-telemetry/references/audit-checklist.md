# Audit checklist: Latitude telemetry

Use when reviewing an existing integration or a pull request that touches observability.

## Bootstrap vs OpenTelemetry

- [ ] **Bootstrap** (`initLatitude` / `init_latitude`) is used for new apps unless multiple span processors are required.
- [ ] **Advanced** path adds `LatitudeSpanProcessor` to an existing provider and calls `registerLatitudeInstrumentations` / `register_latitude_instrumentations` with the **same** `TracerProvider`.

## Correctness

- [ ] `LATITUDE_API_KEY` and `LATITUDE_PROJECT_SLUG` are read from the environment on the server, not hard-coded.
- [ ] LLM client imports occur **after** telemetry bootstrap when patch-based auto-instrumentation requires it.
- [ ] **TypeScript**: `instrumentations` includes every vendor SDK actually used; if spans are missing despite installs, verify explicit **`modules`** (imported client classes) on `registerLatitudeInstrumentations` per [typescript.md](typescript.md).
- [ ] **Python**: `instrumentations` list includes every vendor SDK actually used (OpenAI, Anthropic, and so on).
- [ ] **Vercel AI SDK** code is NOT also registered via `instrumentations: ["openai"]`; it uses `experimental_telemetry: { isEnabled: true }` per call instead.
- [ ] **Python return value** uses item access: `latitude["flush"]()` / `latitude["shutdown"]()`, not attribute access.
- [ ] Short-lived processes call `flush()` / `shutdown()` (or provider `forceFlush()`).

## Context and privacy

- [ ] `capture()` wraps **coarse** boundaries (request, job, agent turn), not every internal helper.
- [ ] Tags and metadata avoid secrets (tokens, raw prompts if policy forbids).
- [ ] Redaction defaults are understood; custom `redact` / `disable_redact` choices are intentional.

## Process lifecycle

- [ ] Initialization runs **once per process** at startup, before LLM clients are constructed.
- [ ] Short-lived processes (CLI, scripts, jobs) call `flush()` / `shutdown()` before exit.
- [ ] Serverless handlers (Lambda, Cloud Run, etc.) `flush()` after LLM work, before returning or suspending.

## Observability of the observability

- [ ] If spans are missing, smart filter settings were checked (`disableSmartFilter` only when justified).
- [ ] For multi-backend setups, spans reach both Latitude and the other vendor per README examples (Datadog, Sentry).
