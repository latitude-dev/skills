# Latitude telemetry — Go

Reference for sending traces from Go applications to Latitude via OTLP HTTP. Loaded from [otlp-fallback.md](otlp-fallback.md) when the host language is Go.

There is no Latitude SDK for Go, and no widely-adopted OpenTelemetry auto-instrumentation for popular Go LLM clients (`openai-go`, `anthropic-sdk-go`). Wrap each LLM call manually with a span that follows `gen_ai.*` semantic conventions.

## Install

```bash
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/sdk/trace \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp \
       go.opentelemetry.io/otel/semconv/v1.26.0
```

## Minimum setup

```go
package telemetry

import (
    "context"
    "os"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func InitLatitude(ctx context.Context, serviceName string) (*sdktrace.TracerProvider, error) {
    apiKey := os.Getenv("LATITUDE_API_KEY")
    projectSlug := os.Getenv("LATITUDE_PROJECT_SLUG")

    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpointURL("https://ingest.latitude.so/v1/traces"),
        otlptracehttp.WithHeaders(map[string]string{
            "Authorization":      "Bearer " + apiKey,
            "X-Latitude-Project": projectSlug,
        }),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName(serviceName),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}
```

Call once at startup; defer `tp.Shutdown(ctx)` for graceful flush.

## Wrap an LLM call

Latitude does not auto-instrument Go LLM clients. Wrap the call yourself:

```go
import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

func ChatWithLatitude(ctx context.Context, model, prompt string) (string, error) {
    tracer := otel.Tracer("app/llm")
    ctx, span := tracer.Start(ctx, "openai.chat",
        trace.WithAttributes(
            attribute.String("gen_ai.system", "openai"),
            attribute.String("gen_ai.operation.name", "chat"),
            attribute.String("gen_ai.request.model", model),
            // Optional Latitude context
            attribute.String("user.id", "user_123"),
            attribute.String("session.id", "session_abc"),
            attribute.String("latitude.tags", `["production"]`),
        ),
    )
    defer span.End()

    resp, err := openaiClient.Chat(ctx, model, prompt)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return "", err
    }

    span.SetAttributes(
        attribute.Int("gen_ai.usage.input_tokens", resp.Usage.PromptTokens),
        attribute.Int("gen_ai.usage.output_tokens", resp.Usage.CompletionTokens),
        attribute.String("gen_ai.response.model", resp.Model),
    )
    return resp.Content, nil
}
```

## Flush before exit

Long-running services rely on the batch processor flushing on its own. CLIs, jobs, and serverless functions must shut down explicitly:

```go
defer func() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := tp.Shutdown(ctx); err != nil {
        log.Printf("latitude shutdown: %v", err)
    }
}()
```

## Common pitfalls

| Mistake | Fix |
| --- | --- |
| Forgetting `tp.Shutdown` in CLIs / Lambda | Spans buffered, never sent. Always defer shutdown. |
| Hardcoding the endpoint | Use `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` if you may switch to self-hosted. |
| Setting attributes after `span.End()` | Dropped silently. Call `SetAttributes` before End or while span is active. |
| Missing `gen_ai.system` | Span lands but Latitude UI does not classify it as an LLM call. |
| Goroutine leaks the parent context | LLM calls run with `context.Background()` — pass the request `ctx` so spans nest under HTTP/handler spans. |
