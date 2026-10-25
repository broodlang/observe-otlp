# observe-otlp

[OpenTelemetry](https://opentelemetry.io) as an [`observe`](../observe) provider for
[Brood](https://broodlang.org): logs, metrics and errors over
[OTLP/HTTP](https://opentelemetry.io/docs/specs/otlp/) with the JSON encoding. One wire
format reaches every backend that speaks OpenTelemetry — a local collector, Grafana,
Honeycomb, Datadog, New Relic, SigNoz, Uptrace — each at an endpoint with an auth header.

## Usage

```brood
(observe/start {:provider (observe-otlp/provider
                            {:endpoint "https://api.honeycomb.io"
                             :headers {"x-honeycomb-team" (os/env "HONEYCOMB_KEY")}
                             :service "myapp"})
                :env "prod" :revision (os/env "FLY_MACHINE_VERSION") :app "myapp"})
(log/add-backend (observe/log-backend {:min-level :info}))
```

Or from the standard variables — `OTEL_EXPORTER_OTLP_ENDPOINT`,
`OTEL_EXPORTER_OTLP_HEADERS` (`k=v,k2=v2`), `OTEL_SERVICE_NAME` — with
`(observe-otlp/from-environment "myapp")`, which is nil (and says so in the log) when no
endpoint is set. The default endpoint is a collector on this machine,
`http://localhost:4318`.

What lands where:

| observe | OTLP | shape |
|---|---|---|
| log record | `/v1/logs` | `LogRecord`: `timeUnixNano`, severity number and text (DEBUG 5, INFO 9, WARN 13, ERROR 17), `body`, the record's `:meta` as attributes |
| error report | `/v1/logs` | a `LogRecord` at ERROR with `exception.type`, `exception.message`, `exception.stacktrace` and its tags and params — OpenTelemetry's exception outside a span, since `observe` has no traces |
| metric | `/v1/metrics` | a gauge; a counter as a monotonic DELTA sum; a timing as a gauge in `ms` (a sample, not a distribution, so no histogram) |

Every batch is reported under one resource: `service.name`, `deployment.environment`,
`service.version` (the revision) and `host.name`. 64-bit integers — times and int
attributes — are decimal strings, as the protobuf JSON mapping writes them.

`:transport` (a `(fn (url body headers) -> response)`) replaces `http/post` for a test; the
suite runs against a recording one and never touches the network.

## Development

`nest fetch` resolves the `observe` dependency. The suite is
`tests/observe-otlp_test.blsp`; `nest format` before committing.
