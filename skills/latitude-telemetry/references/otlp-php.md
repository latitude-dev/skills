# Latitude telemetry — PHP

Reference for sending traces from PHP applications to Latitude via OTLP HTTP. Loaded from [otlp-fallback.md](otlp-fallback.md) when the host language is PHP.

PHP has the thinnest LLM-instrumentation ecosystem of the supported languages. There is no Latitude SDK for PHP, and no widely-adopted OpenTelemetry auto-instrumentation for `openai-php/client` or comparable libraries. Wrap each LLM call manually with a span that follows `gen_ai.*` semantic conventions.

If the user is running PHP-FPM (typical Laravel / Symfony deployment), follow the **flush-per-request** pattern below — without it, spans are buffered and dropped when the worker recycles.

## Install

```bash
composer require open-telemetry/sdk \
                 open-telemetry/exporter-otlp \
                 open-telemetry/transport-http
```

The OpenTelemetry PHP SDK requires either the **`ffi` extension** (recommended for production) or a polyfill. If `composer install` complains about FFI, install the extension via your platform's package manager (`apt install php-ffi`, `brew install php`, etc.). Reference: [opentelemetry.io/docs/languages/php/getting-started](https://opentelemetry.io/docs/languages/php/getting-started/).

## Minimum setup

A bootstrap file loaded by your framework's container or `index.php`:

```php
<?php
use OpenTelemetry\API\Trace\TracerInterface;
use OpenTelemetry\Contrib\Otlp\SpanExporter;
use OpenTelemetry\SDK\Common\Attribute\Attributes;
use OpenTelemetry\SDK\Common\Export\Http\PsrTransportFactory;
use OpenTelemetry\SDK\Resource\ResourceInfo;
use OpenTelemetry\SDK\Trace\SpanProcessor\BatchSpanProcessor;
use OpenTelemetry\SDK\Trace\TracerProvider;

function buildLatitudeTracer(): TracerInterface
{
    $apiKey = getenv('LATITUDE_API_KEY');
    $projectSlug = getenv('LATITUDE_PROJECT_SLUG');

    $transport = (new PsrTransportFactory())->create(
        'https://ingest.latitude.so/v1/traces',
        'application/x-protobuf',
        [
            'Authorization' => 'Bearer ' . $apiKey,
            'X-Latitude-Project' => $projectSlug,
        ],
    );

    $exporter = new SpanExporter($transport);

    $provider = TracerProvider::builder()
        ->addSpanProcessor(BatchSpanProcessor::builder($exporter)->build())
        ->setResource(ResourceInfo::create(Attributes::create(['service.name' => 'my-service'])))
        ->build();

    register_shutdown_function(fn () => $provider->shutdown());

    return $provider->getTracer('app/llm');
}
```

In Laravel, register this in `AppServiceProvider::register` and bind the tracer as a singleton. In Symfony, expose it through the DI container.

## Wrap an LLM call

```php
<?php
use OpenTelemetry\API\Trace\StatusCode;
use OpenTelemetry\API\Trace\TracerInterface;

function chatWithLatitude(TracerInterface $tracer, string $model, string $prompt): string
{
    $span = $tracer->spanBuilder('openai.chat')
        ->setAttribute('gen_ai.system', 'openai')
        ->setAttribute('gen_ai.operation.name', 'chat')
        ->setAttribute('gen_ai.request.model', $model)
        // Optional Latitude context
        ->setAttribute('user.id', 'user_123')
        ->setAttribute('session.id', 'session_abc')
        ->setAttribute('latitude.tags', '["production"]')
        ->startSpan();

    $scope = $span->activate();

    try {
        $response = $openaiClient->chat()->create([
            'model' => $model,
            'messages' => [['role' => 'user', 'content' => $prompt]],
        ]);

        $span->setAttribute('gen_ai.usage.input_tokens', $response->usage->promptTokens);
        $span->setAttribute('gen_ai.usage.output_tokens', $response->usage->completionTokens);
        $span->setAttribute('gen_ai.response.model', $response->model);

        return $response->choices[0]->message->content;
    } catch (\Throwable $e) {
        $span->recordException($e);
        $span->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());
        throw $e;
    } finally {
        $scope->detach();
        $span->end();
    }
}
```

## Flush before request ends (PHP-FPM)

PHP-FPM workers die between requests, killing the batch processor before it flushes. Two options:

1. **`register_shutdown_function`** (already in the bootstrap above) — calls `$provider->shutdown()` when the request finishes.
2. **Force-flush in after-response middleware** — slower per request but guarantees delivery for short responses:
   - Laravel: `terminate(Request $request, Response $response)` in middleware → `$provider->forceFlush();`
   - Symfony: `kernel.terminate` event listener → `$provider->forceFlush();`

For long-running queue workers (`php artisan queue:work`, `php bin/console messenger:consume`), the batch processor flushes on its own timer; just register a SIGTERM handler that calls `shutdown()` so in-flight batches survive a graceful stop.

## Common pitfalls

| Mistake | Fix |
| --- | --- |
| Spans buffered, FPM worker dies before flush | Use `register_shutdown_function` or `forceFlush()` in terminating middleware. |
| Missing `php-ffi` extension on the host | The OTel SDK falls back to a slower polyfill or fails to load — install `php-ffi` for production. |
| Worker pool reuses the same provider across requests after a shutdown | Build a fresh provider per request **or** never call `shutdown()` mid-life — use `forceFlush()` instead. |
| `$scope->detach()` not called in `finally` | Context manager leaks; subsequent spans may parent under the wrong span. Always detach. |
| Missing `gen_ai.system` | Span lands but Latitude UI does not classify it as an LLM call. |
