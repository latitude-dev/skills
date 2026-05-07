# Latitude telemetry — Java

Reference for sending traces from Java applications to Latitude via OTLP HTTP. Loaded from [otlp-fallback.md](otlp-fallback.md) when the host language is Java.

OpenLLMetry has partial Java coverage. Before writing manual spans, check whether a community OpenTelemetry instrumentation exists for the LLM client in use (e.g. LangChain4j ships its own OTel hooks). If none exists for the LLM client, wrap each call manually with a span that follows `gen_ai.*` semantic conventions.

## Install

Maven (`pom.xml`):

```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
    <version>1.42.1</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
    <version>1.42.1</version>
</dependency>
```

Gradle:

```groovy
implementation 'io.opentelemetry:opentelemetry-sdk:1.42.1'
implementation 'io.opentelemetry:opentelemetry-exporter-otlp:1.42.1'
```

Use the latest version — confirm at [Maven Central](https://central.sonatype.com/artifact/io.opentelemetry/opentelemetry-sdk).

## Minimum setup

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;

public final class LatitudeTelemetry {
    public static OpenTelemetry init(String serviceName) {
        var apiKey = System.getenv("LATITUDE_API_KEY");
        var projectSlug = System.getenv("LATITUDE_PROJECT_SLUG");

        var exporter = OtlpHttpSpanExporter.builder()
            .setEndpoint("https://ingest.latitude.so/v1/traces")
            .addHeader("Authorization", "Bearer " + apiKey)
            .addHeader("X-Latitude-Project", projectSlug)
            .build();

        var provider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .setResource(Resource.getDefault().merge(
                Resource.create(Attributes.builder()
                    .put("service.name", serviceName)
                    .build())
            ))
            .build();

        var sdk = OpenTelemetrySdk.builder()
            .setTracerProvider(provider)
            .buildAndRegisterGlobal();

        Runtime.getRuntime().addShutdownHook(new Thread(() ->
            provider.shutdown().join(10, TimeUnit.SECONDS)
        ));

        return sdk;
    }
}
```

Call `LatitudeTelemetry.init("my-service")` once at application start (e.g. a `@PostConstruct` bean in Spring, or `main`).

## Wrap an LLM call

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.context.Scope;

public String chatWithLatitude(String model, String prompt) {
    var tracer = GlobalOpenTelemetry.getTracer("app/llm");
    var span = tracer.spanBuilder("openai.chat")
        .setAttribute("gen_ai.system", "openai")
        .setAttribute("gen_ai.operation.name", "chat")
        .setAttribute("gen_ai.request.model", model)
        // Optional Latitude context
        .setAttribute("user.id", "user_123")
        .setAttribute("session.id", "session_abc")
        .setAttribute("latitude.tags", "[\"production\"]")
        .startSpan();

    try (Scope scope = span.makeCurrent()) {
        var response = openaiClient.chat(model, prompt);
        span.setAttribute("gen_ai.usage.input_tokens", response.getPromptTokens());
        span.setAttribute("gen_ai.usage.output_tokens", response.getCompletionTokens());
        span.setAttribute("gen_ai.response.model", response.getModel());
        return response.getContent();
    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR, e.getMessage());
        throw e;
    } finally {
        span.end();
    }
}
```

## Flush before exit

The `addShutdownHook` in `init` covers most cases. For servlet containers that may not run JVM shutdown hooks (Tomcat, WildFly under certain configurations), wire the SDK shutdown into the framework lifecycle (`@PreDestroy` in Spring, `ServletContextListener.contextDestroyed`, etc.).

## Common pitfalls

| Mistake | Fix |
| --- | --- |
| `tracer.spanBuilder(...).startSpan()` without a `try/finally span.end()` | Span never ends; the batch processor holds it indefinitely. Always end in finally. |
| Skipping `span.makeCurrent()` | HTTP-client / DB instrumentation spans don't nest under the LLM span. |
| Using a raw global `OpenTelemetry.getTracerProvider()` after init when bean lifecycle resets it | Use `GlobalOpenTelemetry.getTracer(...)` and avoid re-initializing the provider per request. |
| Missing `gen_ai.system` | Span lands but Latitude UI does not classify it as an LLM call. |
| Servlet / Spring not invoking shutdown hooks | Add `@PreDestroy` (Spring) or container-specific lifecycle listener that calls `provider.shutdown()`. |
