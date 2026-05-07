# Latitude telemetry — .NET

Reference for sending traces from .NET applications to Latitude via OTLP HTTP. Loaded from [otlp-fallback.md](otlp-fallback.md) when the host language is .NET.

There is no Latitude SDK for .NET, and LLM-specific OpenTelemetry instrumentations are rare. Wrap each LLM call manually with an `Activity` (the .NET name for an OTel span) that follows `gen_ai.*` semantic conventions.

## Install

```bash
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Extensions.Hosting
```

## Minimum setup

`Program.cs` (ASP.NET Core / minimal hosting):

```csharp
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Exporter;

var apiKey = Environment.GetEnvironmentVariable("LATITUDE_API_KEY")!;
var projectSlug = Environment.GetEnvironmentVariable("LATITUDE_PROJECT_SLUG")!;

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("my-service"))
        .AddSource("app/llm") // must match the ActivitySource name below
        .AddOtlpExporter(opt =>
        {
            opt.Endpoint = new Uri("https://ingest.latitude.so/v1/traces");
            opt.Headers = $"Authorization=Bearer {apiKey},X-Latitude-Project={projectSlug}";
            opt.Protocol = OtlpExportProtocol.HttpProtobuf;
        }));
```

Console / non-hosted apps:

```csharp
using OpenTelemetry;

using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddSource("app/llm")
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("my-service"))
    .AddOtlpExporter(opt =>
    {
        opt.Endpoint = new Uri("https://ingest.latitude.so/v1/traces");
        opt.Headers = $"Authorization=Bearer {apiKey},X-Latitude-Project={projectSlug}";
        opt.Protocol = OtlpExportProtocol.HttpProtobuf;
    })
    .Build();
```

The `using` in console apps disposes the provider on exit, which flushes pending spans.

## Wrap an LLM call

```csharp
using System.Diagnostics;
using OpenTelemetry.Trace;

private static readonly ActivitySource LlmActivitySource = new("app/llm");

public async Task<string> ChatWithLatitudeAsync(string model, string prompt)
{
    using var activity = LlmActivitySource.StartActivity("openai.chat", ActivityKind.Client);
    activity?.SetTag("gen_ai.system", "openai");
    activity?.SetTag("gen_ai.operation.name", "chat");
    activity?.SetTag("gen_ai.request.model", model);
    // Optional Latitude context
    activity?.SetTag("user.id", "user_123");
    activity?.SetTag("session.id", "session_abc");
    activity?.SetTag("latitude.tags", "[\"production\"]");

    try
    {
        var response = await _openaiClient.ChatAsync(model, prompt);
        activity?.SetTag("gen_ai.usage.input_tokens", response.Usage.PromptTokens);
        activity?.SetTag("gen_ai.usage.output_tokens", response.Usage.CompletionTokens);
        activity?.SetTag("gen_ai.response.model", response.Model);
        return response.Content;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

The `using` declaration calls `activity.Stop()` when the method returns or throws.

## Flush before exit

The hosted lifetime (ASP.NET Core, generic host) flushes on graceful shutdown. For console apps, `using` covers it. For Azure Functions or AWS Lambda where the host can suspend mid-execution, force-flush before returning the response:

```csharp
var tracerProvider = host.Services.GetRequiredService<TracerProvider>();
tracerProvider.ForceFlush(TimeSpan.FromSeconds(5));
```

## Common pitfalls

| Mistake | Fix |
| --- | --- |
| Forgetting `.AddSource("app/llm")` on the tracer provider | Activities created from your `ActivitySource` are silently dropped. Every custom source name must be registered. |
| Setting tags on `Activity.Current` after the activity ended | No-op. Tag inside the `using` scope only. |
| Returning from an `async` method without `await`ing the LLM call | The `using` disposes the activity before the operation completes; the span has zero duration. Always `await`. |
| Missing `gen_ai.system` | Span lands but Latitude UI does not classify it as an LLM call. |
| Mixing `ActivitySource` instances and forgetting one in `AddSource` | Spans from that source disappear. Use a single source per layer or register all of them. |
