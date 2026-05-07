# Latitude telemetry — Ruby

Reference for sending traces from Ruby applications to Latitude via OTLP HTTP. Loaded from [otlp-fallback.md](otlp-fallback.md) when the host language is Ruby.

There is no Latitude SDK for Ruby, and no widely-adopted OpenTelemetry auto-instrumentation for `ruby-openai` / `anthropic` / similar gems. Wrap each LLM call manually with a span that follows `gen_ai.*` semantic conventions.

The standard `opentelemetry-instrumentation-all` package does instrument Net::HTTP and Faraday, so the underlying HTTP request to OpenAI will produce a span automatically — you still need a parent LLM span to attach `gen_ai.*` attributes and Latitude context.

## Install

```bash
bundle add opentelemetry-sdk opentelemetry-exporter-otlp
```

Or in `Gemfile`:

```ruby
gem 'opentelemetry-sdk'
gem 'opentelemetry-exporter-otlp'
```

## Minimum setup

`config/initializers/latitude_telemetry.rb` (Rails) or any startup file:

```ruby
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'

api_key = ENV.fetch('LATITUDE_API_KEY')
project_slug = ENV.fetch('LATITUDE_PROJECT_SLUG')

OpenTelemetry::SDK.configure do |c|
  c.service_name = 'my-service'
  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new(
        endpoint: 'https://ingest.latitude.so/v1/traces',
        headers: {
          'Authorization' => "Bearer #{api_key}",
          'X-Latitude-Project' => project_slug,
        }
      )
    )
  )
end

at_exit { OpenTelemetry.tracer_provider.shutdown }
```

Or set the standard env vars and skip the explicit exporter config:

```bash
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="https://ingest.latitude.so/v1/traces"
export OTEL_EXPORTER_OTLP_TRACES_HEADERS="Authorization=Bearer YOUR_KEY,X-Latitude-Project=YOUR_SLUG"
```

## Wrap an LLM call

```ruby
TRACER = OpenTelemetry.tracer_provider.tracer('app/llm')

def chat_with_latitude(model:, prompt:)
  TRACER.in_span(
    'openai.chat',
    attributes: {
      'gen_ai.system' => 'openai',
      'gen_ai.operation.name' => 'chat',
      'gen_ai.request.model' => model,
      # Optional Latitude context
      'user.id' => 'user_123',
      'session.id' => 'session_abc',
      'latitude.tags' => '["production"]',
    }
  ) do |span|
    response = openai_client.chat(parameters: {
      model: model,
      messages: [{ role: 'user', content: prompt }],
    })

    span.set_attribute('gen_ai.usage.input_tokens', response.dig('usage', 'prompt_tokens'))
    span.set_attribute('gen_ai.usage.output_tokens', response.dig('usage', 'completion_tokens'))
    span.set_attribute('gen_ai.response.model', response['model'])

    response.dig('choices', 0, 'message', 'content')
  rescue StandardError => e
    span.record_exception(e)
    span.status = OpenTelemetry::Trace::Status.error(e.message)
    raise
  end
end
```

`in_span` automatically ends the span when the block returns or raises.

## Flush before exit

The `at_exit` in setup covers Ruby scripts and Rake tasks. Three runtime variants need extra care:

- **Puma / Unicorn (forked workers)** — initialize OTel **after** workers fork, in `on_worker_boot`. Each worker process needs its own provider.
- **Sidekiq jobs** — register a shutdown callback so spans flush before the worker exits:
  ```ruby
  Sidekiq.configure_server do |config|
    config.on(:shutdown) { OpenTelemetry.tracer_provider.shutdown }
  end
  ```
- **AWS Lambda (ruby runtime)** — call `OpenTelemetry.tracer_provider.force_flush(timeout: 5)` before the handler returns; `at_exit` may not fire reliably.

## Common pitfalls

| Mistake | Fix |
| --- | --- |
| Initializing OTel before Puma forks | Each worker inherits a stale provider. Init in `on_worker_boot` instead. |
| Using `tracer.start_span` without `with_span do` and forgetting to call `span.finish` | Span never ends. Prefer the block form `tracer.in_span("name") do |span| ... end`. |
| Sidekiq exiting before flush | Configure `on(:shutdown)` to call `tracer_provider.shutdown`. |
| Missing `gen_ai.system` | Span lands but Latitude UI does not classify it as an LLM call. |
| Net::HTTP instrumentation creating an extra span outside the LLM span | Confirm the `in_span` block wraps the actual HTTP call so the request span nests underneath. |
