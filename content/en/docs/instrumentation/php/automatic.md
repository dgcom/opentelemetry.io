---
title: Automatic Instrumentation
linkTitle: Automatic
weight: 20
spelling: cSpell:ignore userland phar autoload tracecontext myapp configurator
spelling: cSpell:ignore packagist pecl shortcode unindented democlass autoloading
---

Automatic instrumentation with PHP requires at least PHP 8.0, and the
[OpenTelemetry PHP extension](https://github.com/open-telemetry/opentelemetry-php-instrumentation).
The extension enables registering observer functions (as PHP code) against
classes and methods, and executing those functions before and after the observed
method runs.

{{% alert title="Important" color="warning" %}}Installing the OpenTelemetry
extension by itself will not generate traces. You must also install one or more
[packages](/ecosystem/registry/?component=instrumentation&language=php) for the
frameworks and libraries that you are using, or alternatively write your own.

You also _must_ use
[composer autoloading](https://getcomposer.org/doc/01-basic-usage.md#autoloading),
as this is the mechanism all auto-instrumentation packages use to register
themselves. {{% /alert %}}

## Example

```php
<?php

use OpenTelemetry\API\Common\Instrumentation\CachedInstrumentation;
use OpenTelemetry\API\Trace\Span;
use OpenTelemetry\API\Trace\StatusCode;
use OpenTelemetry\Context\Context;

require 'vendor/autoload.php';

class DemoClass
{
    public function run(): void
    {
        echo 'Hello, world';
    }
}

OpenTelemetry\Instrumentation\hook(
    class: DemoClass::class,
    function: 'run',
    pre: static function (DemoClass $demo, array $params, string $class, string $function, ?string $filename, ?int $lineno) {
        static $instrumentation;
        $instrumentation ??= new CachedInstrumentation('example');
        $span = $instrumentation->tracer()->spanBuilder('democlass-run')->startSpan();
        Context::storage()->attach($span->storeInContext(Context::getCurrent()));
    },
    post: static function (DemoClass $demo, array $params, $returnValue, ?Throwable $exception) {
        $scope = Context::storage()->scope();
        $scope->detach();
        $span = Span::fromContext($scope->context());
        if ($exception) {
            $span->recordException($exception);
            $span->setStatus(StatusCode::STATUS_ERROR);
        }
        $span->end();
    }
);

$demo = new DemoClass();
$demo->run();
```

Here, we provide `pre` and `post` functions, which are executed before and after
`DemoClass::run`. The `pre` function starts and activates a span, and the `post`
function ends it. If an exception was thrown by `DemoClass::run()`, the `post`
function will record it, without affecting exception propagation.

## Installation

The extension can be installed via pecl,
[pickle](https://github.com/FriendsOfPHP/pickle) or
[php-extension-installer](https://github.com/mlocati/docker-php-extension-installer)
(docker specific).

1. Setup development environment. Installing from source requires proper
   development environment and some dependencies:

   {{< tabpane lang=shell >}}

   {{< tab "Linux (apt)" >}}sudo apt-get install gcc make autoconf{{< /tab >}}

   {{< tab "macOS (homebrew)" >}}brew install gcc make autoconf{{< /tab >}}

   {{< /tabpane >}}

2. Build/install the extension. With your environment set up you can install the
   extension:

   {{< tabpane lang=shell >}}

   {{< tab pecl >}}pecl install opentelemetry-beta{{< /tab >}}

<!-- The remaining shortcode lines must be unindented so that tab content is unindented in the generated page -->
<!-- prettier-ignore-start -->
{{< tab pickle >}}
php pickle.phar install opentelemetry
{{< /tab >}}

{{< tab "php-extension-installer (docker)" >}}
install-php-extensions opentelemetry
{{< /tab >}}

{{< /tabpane >}}
<!-- prettier-ignore-end -->

3. Add the extension to your `php.ini` file:

   ```ini
   [opentelemetry]
   extension=opentelemetry.so
   ```

4. Verify that the extension is installed and enabled:

   ```sh
   php -m | grep opentelemetry
   ```

## Zero-code configuration for automatic instrumentation

When used in conjunction with the OpenTelemetry SDK, you can use environment
variables or `php.ini` to configure auto-instrumentation:

```sh
OTEL_PHP_AUTOLOAD_ENABLED=true \
OTEL_SERVICE_NAME=your-service-name \
OTEL_TRACES_EXPORTER=otlp \
OTEL_EXPORTER_OTLP_PROTOCOL=grpc \
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317 \
OTEL_PROPAGATORS=baggage,tracecontext \
php myapp.php
```

## Manual setup for automatic instrumentation

```php
<?php
OpenTelemetry\API\Common\Instrumentation\Globals::registerInitializer(function (Configurator $configurator) {
    $propagator = TraceContextPropagator::getInstance();
    $spanProcessor = new BatchSpanProcessor(/*params*/);
    $tracerProvider = (new TracerProviderBuilder())
        ->addSpanProcessor($spanProcessor)
        ->setSampler(new ParentBased(new AlwaysOnSampler()))
        ->build();

    ShutdownHandler::register([$tracerProvider, 'shutdown']);

    return $configurator
        ->withTracerProvider($tracerProvider)
        ->withPropagator($propagator);
});

//instrumentation libraries can access the configured providers (or a no-op implementation) via `Globals`
$tracer = Globals::tracerProvider()->getTracer('example');
//or, via CachedInstrumentation which uses `Globals` internally
$instrumentation = new CachedInstrumentation('example');
$tracerProvider = $instrumentation->tracer();
```

## Supported libraries and frameworks

Automatic Instrumentation comes with a number of instrumentation libraries for
commonly used PHP libraries. For the full list, see
[instrumentation libraries on packagist](https://packagist.org/search/?query=open-telemetry&tags=instrumentation).

## Next steps

After you have automatic instrumentation configured for your app or service, you
might want to add [manual instrumentation](../manual) to collect custom
telemetry data.
