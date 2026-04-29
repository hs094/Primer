# 03 — OTel SDK (Python)

🔑 Three providers, one OTLP exporter, auto-instrument the framework — done.

Source: https://opentelemetry.io/docs/languages/python/

## Packages
```bash
uv add opentelemetry-api opentelemetry-sdk \
       opentelemetry-exporter-otlp \
       opentelemetry-instrumentation-fastapi \
       opentelemetry-instrumentation-httpx \
       opentelemetry-instrumentation-sqlalchemy
```

## Minimal Tracer Setup
```python
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

resource = Resource.create({"service.name": "api", "service.version": "1.0.0"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))  # endpoint via OTEL_EXPORTER_OTLP_ENDPOINT
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

async def handle(req):
    with tracer.start_as_current_span("handle") as span:
        span.set_attribute("user.id", req.user_id)
        return await do_work()
```

## FastAPI Auto-Instrumentation
```python
from fastapi import FastAPI
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

app = FastAPI()
FastAPIInstrumentor.instrument_app(app)  # spans every route automatically
```

## Metrics + Logs (same shape)
- `MeterProvider` + `PeriodicExportingMetricReader` + `OTLPMetricExporter`
- `LoggerProvider` + `BatchLogRecordProcessor` + `OTLPLogExporter`; attach `LoggingHandler` to stdlib `logging`.

## Gotchas
- ⚠️ `BatchSpanProcessor` drops on shutdown if you don't `provider.shutdown()` — register `atexit` or use FastAPI lifespan.
- ⚠️ Use **gRPC** OTLP in prod (lower overhead); HTTP/proto for sidecars without gRPC.
- 💡 Auto-instrument before manual spans — context propagation depends on it.

## Tags
[[OpenTelemetry]] [[Python]] [[FastAPI]] [[OTLP]]
