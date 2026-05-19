---
name: latitude-telemetry
description: Add or review Latitude Telemetry for LLM apps. Use for Latitude tracing, LLM observability, missing traces, OpenTelemetry/OTLP integration in TypeScript, Python, and other runtimes, including using Latitude MCP to discover telemetry configuration values.
---

# Latitude Telemetry

Add or review Latitude Telemetry without disrupting existing observability. Latitude is OpenTelemetry-based, so compose with the app's current OTel/Sentry/Datadog setup instead of replacing it.

## Workflow

1. **Audit first**
   - Identify languages, package managers, entry points, runtimes, deployment config, and env conventions.
   - Check the package registry for the latest Latitude SDK for the target language; use the current alpha if it is the latest release, and do not copy versions from examples.
   - Check for `LATITUDE_API_KEY` and either `LATITUDE_PROJECT_SLUG` or per-capture project routing. Look in env files, secret-manager references, deployment config, and CI. If missing, direct the user to add real values in the existing secret/config system; add placeholders only to examples/docs.
   - Find existing telemetry: `@opentelemetry/*`, `opentelemetry-*`, `dd-trace`, `@sentry/*`, `sentry-sdk`, `newrelic`, Honeycomb, Jaeger/Tempo/OTLP exporters, LangSmith/Langfuse/Helicone/Phoenix/Traceloop, custom span processors, and `OTEL_*` env vars. Existing SDKs usually initialize first; Latitude initializes second or attaches a `LatitudeSpanProcessor` to the existing provider.
   - Find LLM call sites: OpenAI chat/responses, Anthropic messages, Bedrock, Cohere, Together, Vertex/Google AI, Azure OpenAI, Vercel AI SDK `generateText`/`streamText`, LangChain, LlamaIndex, OpenAI Agents, LiteLLM, CrewAI, etc. Trace from route/job/CLI/agent entry points to the actual model calls. Note streaming paths; consume streams inside the capture boundary.
   - If the request is not clearly covered here, consult the Latitude docs (`https://docs.latitude.so/llms.txt`, especially `telemetry/*`) and/or the telemetry package implementation in `github.com/latitude-dev/latitude-llm/packages/telemetry/*`.

2. **Group use cases**
   - Group related prompts, tools, retrieval, and model calls by final goal, not by file.
   - For each group, record: entry point, LLM calls, available user/session IDs, useful tags/metadata, streaming behavior, and project routing.
   - Prefer one `capture()` around the request, job, or agent turn. Add nested captures only for meaningful sub-boundaries.

3. **Clarify gaps before planning**
   - Ask only material questions the codebase cannot answer: where traces should appear in Latitude, who owns missing secrets/config, whether existing observability must stay unchanged, ambiguous use-case boundaries, or approval for broad refactors.
   - Assume no Latitude SDK or OpenTelemetry knowledge. Do not ask the user to choose between SDKs, processors, exporters, env-var schemes, instrumentation keys, or OTLP wiring unless they have shown that expertise.
   - When a technical choice is needed, infer the best option from the codebase, explain the user-visible outcome and tradeoff in plain language, and ask for confirmation. Keep implementation details for the plan.
   - Ask one question at a time, in priority order, and stop until answered. Never combine an unresolved material question with a request for approval.
   - State low-risk assumptions in the plan instead of asking.

4. **Plan, then wait**
   After material clarifications are resolved, including the preliminary Latitude MCP install/connect question when MCP is unavailable, present a concise plan and wait for explicit approval before installing packages, editing files, or changing configuration. Explain each change in plain language. Approval must be unambiguous (`yes`, `approved`, `go ahead`). If the user requests changes or answers a missed clarification, revise the plan and ask again. Do not include unresolved clarification questions or optional MCP setup offers inside the plan; ask and resolve them before this step.

   ```text
   Plan
   - What Latitude will add: capture LLM requests/responses, token/model details, errors, latency, and user/session context where available.
   - Decisions/assumptions: ... (resolved clarifications, defaults chosen, and why they are safe)
   - Existing telemetry: ... (what is already present and whether it will be preserved)
   - LLM integrations found: ... (which model calls will be traced)
   - Capture boundaries: ... (which request/job/agent turn becomes one trace, and why)
   - Integration approach: ... (SDK bootstrap / existing OTel processor / generic OTLP, explained in plain language)
   - Env/config needed: ... (which values are needed, what they do, where to find them, and where placeholders/docs will be added)
   - Files to change: ... (what each file change accomplishes)
   - Verification: ... (checks to run and how to confirm traces arrive)

   Reply `go ahead` to approve this plan.
   ```

5. **Implement only after approval**
   - Do not act on implied approval or silence; ask when intent is unclear.
   - Follow existing patterns for config validation, module layout, logging, tests, and package management. Let the lockfile capture the resolved SDK version.
   - Keep changes targeted to telemetry: packages, Latitude initialization, capture boundaries, provider telemetry, and env examples/docs. Get separate explicit permission for broad refactors such as changing module type, switching build systems, reorganizing app structure, replacing telemetry vendors, changing framework/runtime config, or rewriting LLM abstractions.
   - Never inline real secrets. Use env vars or the project's secret manager. If Latitude MCP tools are available, use them to confirm/create project or key metadata; otherwise ask where missing secrets should be managed.
   - Initialize Latitude once at startup/module scope, before the first LLM call when possible. Avoid per-request SDK instances.
   - Preserve current observability; do not remove span processors/exporters unless the user explicitly approves.
   - Run formatter/typecheck/lint/tests. If credentials and a safe path exist, run one representative LLM flow per use-case group and flush before exit. For missing traces, check env values, project routing, initialization order, instrumentation registration, smart filtering, stream consumption inside `capture()`, and process exit before flush.

## Latitude MCP-assisted configuration

Use this section when adding Latitude telemetry and configuration values are missing or ambiguous. The Latitude MCP is a remote OAuth-authenticated MCP server at `https://api.latitude.so/v1/mcp` that can expose Latitude workspace data and actions to the agent. It is not required for telemetry, but when it is connected it should be used to reduce user back-and-forth.

- **Check MCP availability first.** Before asking the user for Latitude project/API-key details, inspect the connected MCP tools/servers available in the current agent harness. If a Latitude MCP server is available and authenticated, use it to discover organization/project metadata and to help prepare telemetry configuration.
- **Offer MCP installation as a preliminary clarification, before the plan.** If the Latitude MCP is not connected, stop before presenting the implementation plan and ask whether the user wants to install/connect it so the agent can automatically discover projects and help fill configurable telemetry variables. Briefly explain that the Latitude MCP gives the agent OAuth-scoped access to their Latitude workspace, including projects, keys, traces, annotations, scores, searches, issues, datasets, and other Latitude resources; connected agents can be revoked under **Settings → Keys → OAuth Keys**. Do not install or configure the MCP without explicit approval. Do not bundle this MCP question into the implementation plan or approval request. If the user declines, continue with the normal manual configuration flow and then present the implementation plan.
- **Use MCP to fill non-secret config.** Prefer MCP-provided project data to identify the correct `LATITUDE_PROJECT_SLUG` when the user has already indicated, or the repo clearly implies, which Latitude project should receive traces. If multiple plausible projects exist, present the options and ask the user to choose one.
- **Use MCP for secret creation/metadata only when safe.** If the Latitude MCP exposes API-key management, use it only after user approval and only to create or identify the needed key metadata. Do not print real API key values in chat. Put secrets directly into the project's existing secret manager only when the harness/tooling supports doing so safely; otherwise add placeholders to env examples/docs and tell the user where to store the real value.
- **Do not ask for values the MCP can answer.** If the MCP can list projects, infer slugs, or confirm existing key names, do that before asking the user. Ask only for decisions MCP cannot know, such as which project should receive traces when ambiguous, whether to create a new API key, or where secrets should be stored.
- **Keep MCP separate from app telemetry.** MCP helps configure Latitude; it does not trace the target app's LLM calls. The app still needs the telemetry SDK or OTLP exporter configured with `LATITUDE_API_KEY` and `LATITUDE_PROJECT_SLUG` or equivalent OTLP headers.

## Configuration values

When asking the user to provide config, explain what each value is and where to find it:

- `LATITUDE_API_KEY`: authenticates uploads to Latitude. Find or create it in Latitude under **Settings → API Keys**.
- `LATITUDE_PROJECT_SLUG`: chooses which Latitude project receives traces. In the Latitude app, open the project; the slug appears in the sidebar title section. It is the short project identifier, not the display name.
- Generic OTLP setups may encode the same values as `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://ingest.latitude.so/v1/traces` and `OTEL_EXPORTER_OTLP_TRACES_HEADERS=Authorization=Bearer <api-key>,X-Latitude-Project=<project-slug>`.

Never ask for real secret values in chat if the project has an existing secret manager. Ask where the user wants them stored, and add placeholders only to env examples/docs.

## TypeScript

Install the latest `@latitude-data/telemetry` with the project's package manager. Initialize existing Sentry/Datadog/New Relic/Honeycomb/custom OTel first, then Latitude.

```ts
import OpenAI from "openai";
import { Latitude } from "@latitude-data/telemetry";

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  project: process.env.LATITUDE_PROJECT_SLUG!,
  instrumentations: { openai: OpenAI },
});

await latitude.ready; // optional, use when first-call coverage matters
```

Use the project's real env validation; the snippet only shows the SDK shape.

Supported instrumentation keys: `openai`, `openai-agents`, `anthropic`, `bedrock`, `cohere`, `langchain`, `llamaindex`, `togetherai`, `vertexai`, `aiplatform`. Pass the same SDK module object the app imports. For Anthropic and most namespace packages, prefer `import * as AnthropicSDK from "@anthropic-ai/sdk"` then `instrumentations: { anthropic: AnthropicSDK }`.

Special cases:

- **Vercel AI SDK:** initialize Latitude without instrumentations; set `experimental_telemetry.isEnabled: true` on each `generateText`, `streamText`, etc. call.
- **Custom existing OTel:** add `new LatitudeSpanProcessor(apiKey, project)` beside existing processors and call `await registerLatitudeInstrumentations({ instrumentations, tracerProvider })`.

Use `capture(name, async () => { ... }, { userId, sessionId, tags, metadata, project })` at use-case boundaries. `project` overrides the constructor default for multi-project routing. `capture()` adds context to instrumented spans; it does not create LLM spans by itself.

For short-lived scripts/jobs, call `await latitude.flush()` or `await latitude.shutdown()` before exit. Do not call `shutdown()` per request in long-lived servers.

## Python

Requires Python 3.11+. Install the latest `latitude-telemetry` with the project's package manager.

```python
import os
import openai
from latitude_telemetry import Latitude

latitude = Latitude(
    api_key=os.environ["LATITUDE_API_KEY"],
    project=os.environ["LATITUDE_PROJECT_SLUG"],
    instrumentations={"openai": openai},
)
```

If an OpenTelemetry provider is already registered, `Latitude(...)` attaches to it. For custom setups, add `LatitudeSpanProcessor` to the existing provider and call `register_latitude_instrumentations(instrumentations={...}, tracer_provider=provider)`.

Supported keys include `openai`, `openai-agents`, `anthropic`, `bedrock`, `cohere`, `langchain`, `llamaindex`, `togetherai`, `vertexai`, and `aiplatform`. Python also supports `aleph_alpha`, `crewai`, `dspy`, `google_generativeai`, `groq`, `haystack`, `litellm`, `mistralai`, `ollama`, `replicate`, `sagemaker`, `transformers`, and `watsonx`. Pass imported module objects, not string lists.

Use `capture()` as a wrapper with snake_case options, especially when context is per request:

```python
from latitude_telemetry import capture

def run_agent(user_id: str, session_id: str):
    return capture(
        "support-agent-run",
        lambda: agent.run(),
        {"user_id": user_id, "session_id": session_id, "tags": ["support"]},
    )
```

For short-lived processes, call `latitude.flush()` or `latitude.shutdown()` before exit. Do not call `shutdown()` per request in long-lived services.

## Other targets

- **Generic OTLP / other languages:** send traces to `https://ingest.latitude.so/v1/traces` with `Authorization: Bearer <LATITUDE_API_KEY>` and `X-Latitude-Project: <LATITUDE_PROJECT_SLUG>`. For full model/token/message details, ensure LLM spans follow OpenTelemetry GenAI semantic conventions (`gen_ai.*` attributes).
- **Claude Code / OpenClaw:** keep agent-tool telemetry separate from app SDK instrumentation. Ask before installing hooks/plugins because prompts, responses, and tool I/O can be sent to Latitude. Claude Code: `npx -y @latitude-data/claude-code-telemetry install` (full-content only). OpenClaw: `npx -y @latitude-data/openclaw-telemetry-cli install`; offer `--no-content` for structural-only telemetry.
