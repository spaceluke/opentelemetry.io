---
title: Frontend
cSpell:ignore: typeof
---

The frontend is responsible to provide a UI for users, as well as an API
leveraged by the UI or other clients. The application is based on
[Next.JS](https://nextjs.org/) to provide a React web-based UI and API routes.

[Frontend source](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/frontend/)

## Server Instrumentation

It is recommended to use a Node required module when starting your Node.js
application to initialize the SDK and auto-instrumentation. When initializing
the OpenTelemetry Node.js SDK, you optionally specify which auto-instrumentation
libraries to leverage, or make use of the `getNodeAutoInstrumentations()`
function which includes most popular frameworks. The
`utils/telemetry/Instrumentation.js` file contains all code required to
initialize the SDK and auto-instrumentation based on standard
[OpenTelemetry environment variables](/docs/specs/otel/configuration/sdk-environment-variables/)
for OTLP export, resource attributes, and service name.

```javascript
const FrontendTracer = async () => {
  const { ZoneContextManager } = await import('@opentelemetry/context-zone');

  let resource = new Resource({
    [SEMRESATTRS_SERVICE_NAME]: NEXT_PUBLIC_OTEL_SERVICE_NAME,
  });
  const detectedResources = detectResourcesSync({
    detectors: [browserDetector],
  });
  resource = resource.merge(detectedResources);

  const provider = new WebTracerProvider({
    resource,
    spanProcessors: [
      new SessionIdProcessor(),
      new BatchSpanProcessor(
        new OTLPTraceExporter({
          url:
            NEXT_PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT ||
            'http://localhost:4318/v1/traces',
        }),
        {
          scheduledDelayMillis: 500,
        },
      ),
    ],
  });

  const contextManager = new ZoneContextManager();

  provider.register({
    contextManager,
    propagator: new CompositePropagator({
      propagators: [
        new W3CBaggagePropagator(),
        new W3CTraceContextPropagator(),
      ],
    }),
  });

  registerInstrumentations({
    tracerProvider: provider,
    instrumentations: [
      getWebAutoInstrumentations({
        '@opentelemetry/instrumentation-fetch': {
          propagateTraceHeaderCorsUrls: /.*/,
          clearTimingResources: true,
          applyCustomAttributesOnSpan(span) {
            span.setAttribute('app.synthetic_request', IS_SYNTHETIC_REQUEST);
          },
        },
      }),
    ],
  });
};
```

Node required modules are loaded using the `--require` command line argument.
This can be done in the `scripts.start` section of `package.json` and starting
the application using `npm start`.

```json
"scripts": {
  "start": "node --require ./Instrumentation.js server.js",
},
```

## Traces

### Span Exceptions and status

You can use the span object's `recordException` function to create a span event
with the full stack trace of a handled error. When recording an exception also
be sure to set the span's status accordingly. You can see this in the catch
block of the `NextApiHandler` function in the
`utils/telemetry/InstrumentationMiddleware.ts` file.

```typescript
span.recordException(error as Exception);
span.setStatus({ code: SpanStatusCode.ERROR });
```

### Create new spans

New spans can be created and started using
`Tracer.startSpan("spanName", options)`. Several options can be used to specify
how the span can be created.

- `root: true` will create a new trace, setting this span as the root.
- `links` are used to specify links to other spans (even within another trace)
  that should be referenced.
- `attributes` are key/value pairs added to a span, typically used for
  application context.

```typescript
span = tracer.startSpan(`HTTP ${method}`, {
  root: true,
  kind: SpanKind.SERVER,
  links: [{ context: syntheticSpan.spanContext() }],
  attributes: {
    'app.synthetic_request': true,
    [SEMATTRS_HTTP_TARGET]: target,
    [SEMATTRS_HTTP_STATUS_CODE]: response.statusCode,
    [SEMATTRS_HTTP_METHOD]: method,
    [SEMATTRS_HTTP_USER_AGENT]: headers['user-agent'] || '',
    [SEMATTRS_HTTP_URL]: `${headers.host}${url}`,
    [SEMATTRS_HTTP_FLAVOR]: httpVersion,
  },
});
```

## Browser Instrumentation

The web-based UI that the frontend provides is also instrumented for web
browsers. OpenTelemetry instrumentation is included as part of the Next.js App
component in `pages/_app.tsx`. Here instrumentation is imported and initialized.

```typescript
import FrontendTracer from '../utils/telemetry/FrontendTracer';

if (typeof window !== 'undefined') FrontendTracer();
```

The `utils/telemetry/FrontendTracer.ts` file contains code to initialize a
TracerProvider, establish an OTLP export, register trace context propagators,
and register web specific auto-instrumentation libraries. Since the browser will
send data to an OpenTelemetry Collector that will likely be on a separate
domain, CORS headers are also setup accordingly.

As part of the changes to carry over the `synthetic_request` attribute flag for
the backend services, the `applyCustomAttributesOnSpan` configuration function
has been added to the `instrumentation-fetch` library custom span attributes
logic that way every browser-side span will include it.

```typescript
import {
  CompositePropagator,
  W3CBaggagePropagator,
  W3CTraceContextPropagator,
} from '@opentelemetry/core';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { SimpleSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { getWebAutoInstrumentations } from '@opentelemetry/auto-instrumentations-web';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME } from '@opentelemetry/semantic-conventions';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const FrontendTracer = async () => {
  const { ZoneContextManager } = await import('@opentelemetry/context-zone');

  const provider = new WebTracerProvider({
    resource: new Resource({
      [SEMRESATTRS_SERVICE_NAME]: process.env.NEXT_PUBLIC_OTEL_SERVICE_NAME,
    }),
    spanProcessors: [new SimpleSpanProcessor(new OTLPTraceExporter())],
  });

  const contextManager = new ZoneContextManager();

  provider.register({
    contextManager,
    propagator: new CompositePropagator({
      propagators: [
        new W3CBaggagePropagator(),
        new W3CTraceContextPropagator(),
      ],
    }),
  });

  registerInstrumentations({
    tracerProvider: provider,
    instrumentations: [
      getWebAutoInstrumentations({
        '@opentelemetry/instrumentation-fetch': {
          propagateTraceHeaderCorsUrls: /.*/,
          clearTimingResources: true,
          applyCustomAttributesOnSpan(span) {
            span.setAttribute('app.synthetic_request', 'false');
          },
        },
      }),
    ],
  });
};

export default FrontendTracer;
```

## Metrics

TBD

## Logs

TBD

## Baggage

OpenTelemetry Baggage is leveraged in the frontend to check if the request is
synthetic (from the load generator). Synthetic requests will force the creation
of a new trace. The root span from the new trace will contain many of the same
attributes as an HTTP request instrumented span.

To determine if a Baggage item is set, you can leverage the `propagation` API to
parse the Baggage header, and leverage the `baggage` API to get or set entries.

```typescript
const baggage = propagation.getBaggage(context.active());
if (baggage?.getEntry("synthetic_request")?.value == "true") {...}
```
