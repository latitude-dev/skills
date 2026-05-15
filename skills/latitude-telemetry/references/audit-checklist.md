# Audit checklist: Latitude telemetry

Use when reviewing an existing integration or a pull request that touches observability.

## Bootstrap shape

- [ ] **Class-based entry point** is used: `new Latitude({...})` (TS) or `Latitude(...)` (Python). Legacy `initLatitude` / `init_latitude` wrappers are acceptable in older code but new init should use the class.
- [ ] When the host app already builds a `TracerProvider`, it's passed via `tracerProvider` (TS) / `tracer_provider` (Py) — not duplicated.
- [ ] Lower-level path (`LatitudeSpanProcessor` + `registerLatitudeInstrumentations`) is only used when the class's auto-detection doesn't fit; otherwise the class is shorter and less error-prone.

## Version pinning

- [ ] The package manifest pins to an **exact** pre-release version, not a floating tag (`@alpha`, `--pre`, `^`, `~`). Run `npm view @latitude-data/telemetry@alpha version` / `pip index versions latitude-telemetry --pre` to check whether the pinned version is current.
- [ ] Lockfile (`package-lock.json` / `pnpm-lock.yaml` / `uv.lock` / `poetry.lock`) captures the same exact version.

## Correctness

- [ ] `LATITUDE_API_KEY` is read from the environment on the server, not hard-coded.
- [ ] `LATITUDE_PROJECT_SLUG` is set in env when the app uses one project; for multi-project apps, every `capture()` either sets its own `projectSlug` or relies on an OTEL resource attribute (see [project-scoping.md](project-scoping.md)).
- [ ] LLM client imports occur **after** telemetry bootstrap when patch-based auto-instrumentation requires it.
- [ ] **TypeScript**: `await latitude.ready` is called before the first LLM call.
- [ ] **TypeScript — `instrumentations` shape is post-`alpha.11`.** The value MUST be a plain object mapping integration names to LLM SDK modules — e.g. `{ openai: OpenAI, anthropic: AnthropicSDK }`. The bare string-array form (`instrumentations: ["openai"]`) is **removed in `3.0.0-alpha.11`+** and throws at register time. Mechanical rewrite:

  ```diff
  - instrumentations: ["openai"]
  + // ensure `import OpenAI from "openai"` is at the top
  + instrumentations: { openai: OpenAI }
  ```

  If the installed version is `< 3.0.0-alpha.11`, this is also an upgrade gate — bump the pin AND migrate every `instrumentations: [...]` call site in the same PR (see SKILL.md Rule 6 and Step 2's "Upgrading an existing TypeScript install" sub-section). Leaving the old call shape with the new version will throw at bootstrap; leaving the old version pinned silently fails the day Renovate or CI bumps it.
- [ ] **Python**: `instrumentations` list includes every vendor SDK actually used (OpenAI, Anthropic, and so on). Python keeps the string-identifier API; only TypeScript moved to the object form.
- [ ] **Vercel AI SDK** code is NOT also registered via an `instrumentations` entry like `openai: OpenAI`; it uses `experimental_telemetry: { isEnabled: true }` per call instead.
- [ ] **Python class API**: `latitude.flush()` / `latitude.shutdown()` (attribute access). If code calls `latitude["flush"]()` on a `Latitude(...)` instance, that's wrong — item access only applies to the legacy `init_latitude(...)` dict return value.
- [ ] Short-lived processes call `flush()` / `shutdown()` (or provider `forceFlush()`).

## Context and privacy

- [ ] `capture()` wraps **coarse** boundaries (request, job, agent turn), not every internal helper.
- [ ] Tags and metadata avoid secrets (tokens, raw prompts if policy forbids).
- [ ] Redaction defaults are understood; custom `redact` / `disable_redact` choices are intentional.

## Process lifecycle

- [ ] Initialization runs **once per process** at startup, before LLM clients are constructed.
- [ ] Short-lived processes (CLI, scripts, jobs) call `flush()` / `shutdown()` before exit.
- [ ] Serverless handlers (Lambda, Cloud Run, etc.) `flush()` after LLM work, before returning or suspending.

## Multi-project routing (if applicable)

- [ ] Only **one** `Latitude` instance is created per process — even when the app emits to several projects. Two instances will warn about provider attachment and double-process spans; use per-capture `projectSlug` / `project_slug` instead.
- [ ] Secondary project slugs used in `capture({ projectSlug })` actually exist in the org behind `LATITUDE_API_KEY`. Unknown slugs are silently rejected at ingest (the OTel exporter logs the 400 at `diag.WARN`, but the UI just shows nothing).
- [ ] When using bare OpenTelemetry with the OTEL resource attribute `latitude.project`, the resource is set on the `TracerProvider` (not as a span attribute by mistake — see [project-scoping.md](project-scoping.md)).

## Observability of the observability

- [ ] If spans are missing, smart filter settings were checked (`disableSmartFilter` only when justified).
- [ ] For multi-backend setups, spans reach both Latitude and the other vendor per README examples (Datadog, Sentry).
