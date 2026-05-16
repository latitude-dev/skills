# Project scoping: route spans to multiple Latitude projects

Use when a single process should emit traces to more than one Latitude project — for example, several agents sharing a runtime, or a backend whose features map to different projects. The default install (one SDK instance, one `project` in the constructor) handles the single-project case and stops here.

> The SDK option was renamed from `projectSlug` / `project_slug` to `project` in `@latitude-data/telemetry` ≥ `3.0.0-alpha.12` and `latitude-telemetry` ≥ `3.0.0a8`. The legacy names still work and log a deprecation warning. The HTTP header (`X-Latitude-Project`), the span attribute (`latitude.project`), and the env var convention (`LATITUDE_PROJECT_SLUG`) are unchanged.

Source of truth: [docs.latitude.so/telemetry/project-scoping](https://docs.latitude.so/telemetry/project-scoping). This file is a working-developer summary; if the page has drifted, the page wins.

## Resolution chain (server-side)

Every span the ingest endpoint accepts is routed to exactly one project. The server resolves the project per span, in this order:

| Priority | Source | Set by |
| --- | --- | --- |
| 1 | Span attribute `latitude.project` | `capture({ project })` (TS) / `capture({"project": ...})` (Py); or set manually on the span |
| 2 | OTEL resource attribute `latitude.project` | The OpenTelemetry `Resource` on the `TracerProvider` |
| 3 | `X-Latitude-Project` HTTP header | The SDK constructor's `project` option (forwarded automatically) |

Highest wins. A span with no slug at any level **and** no header default is rejected by the ingest endpoint with HTTP 400. A span pointing at a slug that does not exist in the org behind the API key is also rejected.

## Make sure the projects exist *before* writing code

Every slug you pass to `capture({ project })` (or set via header / resource attribute) must already exist in the org behind `LATITUDE_API_KEY` — unknown slugs are silently rejected at ingest. So whatever scoping pattern you pick below, the prerequisite is the same: the projects need to be real.

Two ways to get there during the install conversation:

- **Latitude MCP server (recommended for multi-project setups).** Once installed, the MCP lets the agent call `listProjects` to see what's already there and `createProject` to add missing ones — no leaving the editor. Per-client install commands and the create-project workflow are in [mcp-setup.md](mcp-setup.md). Detect the user's client first (Claude Code CLI, Claude Desktop, Cursor, Codex, Gemini CLI, Zed, OpenCode, …) and use the matching install block; don't guess.
- **Manual creation in the console.** [https://console.latitude.so](https://console.latitude.so) → **New project**, one per slug. The user pastes each slug back into the chat.

Either path is fine. Don't gate the install on MCP — but do recommend it when there are several projects to create, because the alternative is the user click-creating each one and pasting the slug back.

When asking the user how many projects they want, present the trade-off and let them decide — **do not push one option over the other**:

- **One project** fits when the agents are doing *similar work* (e.g. a flaggers system where multiple sub-agents flag content the same way, or a main agent with helpers all aimed at the same outcome). Similar-shaped traces help Latitude's pattern detection and issue search.
- **Multiple projects** fit when the agents have *very different goals* (e.g. a researcher and a summarizer, a customer chat assistant and a nightly batch job). Mixing very different trace shapes in one project makes pattern detection noisier; splitting keeps each project's signal clean.

If the user is uncertain, repeat the "similar work vs. different goals" framing rather than picking for them.

## When to use which pattern

| Situation | Pattern |
| --- | --- |
| App emits to one Latitude project. | Constructor `project`. Nothing else needed. |
| Several agents in one runtime, each owns a project. | Per-capture `project` override. Leave the constructor slug as a fallback (or unset). |
| Background job vs. user-facing route in same service, different projects. | Per-capture override on the job, constructor default for routes. |
| Non-TS/Python service (Go, Java, …) emitting to one project. | `X-Latitude-Project` header on the OTLP exporter. (Default for the [otlp-fallback](otlp-fallback.md) path.) |
| Non-TS/Python service emitting to several projects from one provider. | `latitude.project` as a resource attribute, no header. Lets one OTLP exporter feed many projects without per-batch header juggling. |

## TypeScript

```typescript
import OpenAI from "openai";
import { Latitude, capture } from "@latitude-data/telemetry";

const latitude = new Latitude({
  apiKey: process.env.LATITUDE_API_KEY!,
  project: process.env.LATITUDE_PROJECT_SLUG!, // default for spans without a per-capture override
  instrumentations: { openai: OpenAI },
});

await latitude.ready;
const client = new OpenAI();

// Inherits the constructor's project.
await capture("default-route", async () => {
  await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: "Hi" }],
  });
});

// Routed to a different project than the constructor default.
await capture(
  "evaluation-batch",
  async () => {
    await client.chat.completions.create({
      model: "gpt-4o-mini",
      messages: [{ role: "user", content: "Evaluate this output." }],
    });
  },
  { project: "evaluation-runs" },
);

await latitude.shutdown();
```

You can also omit `project` from the constructor entirely — every `capture()` then has to set its own.

## Python

```python
import openai
from openai import OpenAI

from latitude_telemetry import Latitude, capture

latitude = Latitude(
    api_key=os.environ["LATITUDE_API_KEY"],
    instrumentations={"openai": openai},
    # No project — every capture must set its own.
)

client = OpenAI()

@capture("full-stack-agent-run", {"project": "full-stack-agent"})
def run_full_stack_agent():
    client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Plan a feature."}],
    )

@capture("call-summariser-run", {"project": "call-summariser"})
def run_call_summariser():
    client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Summarize this call."}],
    )

run_full_stack_agent()
run_call_summariser()
latitude.flush()
```

Same shape with `project` set both on the constructor (default) and on individual `capture(...)` calls (override) is also valid — the override wins for those captures.

## Bare OpenTelemetry (no Latitude SDK)

Useful for languages without a first-class Latitude SDK, or for advanced users routing one OTLP exporter to many projects.

Resource attribute (recommended for multi-project setups):

```typescript
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { resourceFromAttributes } from "@opentelemetry/resources";
import { BatchSpanProcessor, NodeTracerProvider } from "@opentelemetry/sdk-trace-node";

const provider = new NodeTracerProvider({
  resource: resourceFromAttributes({
    "service.name": "my-service",
    "latitude.project": "primary", // default project for spans emitted by this provider
  }),
  spanProcessors: [
    new BatchSpanProcessor(
      new OTLPTraceExporter({
        url: "https://ingest.latitude.so/v1/traces",
        headers: { Authorization: `Bearer ${process.env.LATITUDE_API_KEY!}` },
        // No X-Latitude-Project header — routing comes from the resource attribute.
      }),
    ),
  ],
});
provider.register();
```

A span emitted by this provider that **also** carries `latitude.project` as a span attribute will route by the span attribute (priority 1), not the resource (priority 2). Useful for one-off overrides.

## OTLP `partial_success` contract

Important when handling mixed batches:

| Outcome | HTTP | Body |
| --- | --- | --- |
| All spans accepted | 200 | `{}` |
| Mixed (some rejected) | 200 | `{ partialSuccess: { rejectedSpans, errorMessage } }` |
| All spans rejected | 400 | `{ code, message }` (`google.rpc.Status` shape — **not** `partialSuccess`) |
| Empty batch | 202 | `{}` |

A standard OTel exporter will log a partial-success warning at `diag.WARN` level; turn that on if traces seem to vanish silently. The 400 body includes a direct link to the project-scoping docs in `message`.

## Pitfalls

- **Slug must exist in the org behind `LATITUDE_API_KEY`.** Unknown slug → span is silently rejected (well, not silently — the exporter logs the 400 — but the trace never appears). Verify by reading the exporter's diag warnings, not by checking the UI alone.
- **Span attr wins over header default.** A `capture({ project })` for an unknown slug is **not** rescued by the constructor's `project`. The fallback only kicks in when the span carries no slug at all.
- **Don't run two SDK instances to reach two projects.** The OTel context manager is global; a second `new Latitude({...})` will warn about provider attachment and double-process spans. Use one SDK instance + per-capture `project` instead.
- **Rate limiting is org-wide, not per-project.** The ingest endpoint shares one bucket across all projects in an org under the same API key (`ratelimit:ingest:traces:{orgId}:{apiKeyId}`). Multi-project apps share that allowance.

## When this skill should stop and ask the user

- The user has not confirmed the secondary project slug(s) exist in their org. Don't guess slugs — verify with `listProjects` via the [MCP](mcp-setup.md) if installed, fall back to a verification curl otherwise, or just ask. Offer the MCP install path when several projects still need to be created.
- The user is unsure whether two features should share one project or split. Project-scoping is a product decision; surface the trade-off rather than picking for them.
- The MCP `listProjects` call returns a list that doesn't include slugs the user said exist. That usually means the OAuth session bound to the wrong org — surface this and ask before assuming the slugs are gone. (See the wrong-org OAuth pitfall in [mcp-setup.md](mcp-setup.md#pitfalls).)
