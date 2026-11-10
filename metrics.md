# Metrics

Logs answer "what happened in this request". **Metrics** answer "how much and how fast, over time": numbers sampled
every 15 seconds and kept as time series for **14 days**. You read them in **Grafana**, the same place as the logs:
<https://logs.wallet.shatori.com> (folder _Internship_).

```
backends: GET /metrics on port 9464 (internal) ──┐
host CPU / RAM / disk / network ─────────────────┼──► Alloy (every 15 s) ──► Prometheus (14 days) ──► Grafana
CPU / memory of every backend container ─────────┘
```

All three backends expose the same metrics with the same names, so every dashboard works for every service. rates-service and
treasury-service add one metric each (see _What is collected_).

## What is collected

Every series carries `service` (`wallet-core`, `rates-service`, `treasury-service`) and `env` (`staging`,
`kvitka-uat`, `wisla-uat`; `local` on your machine).

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `http_requests_total` | counter | `operation`, `status` | HTTP requests, except `/health`, `/metrics` and CORS preflights |
| `http_request_duration_seconds` | histogram | `operation` | Request duration; buckets 5 ms … 10 s |
| `graphql_errors_total` | counter | `operation`, `code` | Failed GraphQL requests by the error code the client sees, each code once per request |
| `graphql_ws_connections` | gauge | — | Open GraphQL websocket connections (subscriptions) |
| `events_published_total` | counter | `type` | Events published through `EventPublisher` |
| `process_*`, `nodejs_*` | default | — | CPU, memory, heap, event-loop lag, GC, process start time |

- **`operation`** is the operation type and its **first root field** (`query me`, `mutation convert`), or the route
  of a REST request (`POST /diagnostics/ping`). The operation name the client sends (`query Me`) is not used: any
  client, logged in or not, can invent names, and each new name would be a new series. Field names come from the
  schema, so their number is bounded. GraphQL that failed to parse or validate, requests that matched no route, and
  values past a cap of 100 all go into one bucket, `(unknown)`. The access log still shows the client's name
  (`query Me`), so search logs by that and metrics by the root field.
- **`code`** is the code of the response: `UNAUTHENTICATED`, `FORBIDDEN`, `BAD_USER_INPUT`, a domain code such as
  `INVALID_CREDENTIALS`, or `INTERNAL_SERVER_ERROR`. Every unexpected error reaches the client as
  `INTERNAL_SERVER_ERROR`, so **`code="INTERNAL_SERVER_ERROR"` means a server-side failure**; every other code is
  an expected, client-caused error.
- **Websocket operations** (graphql-ws) count in `graphql_errors_total` the same way (`code` and `operation` as over
  HTTP, `GRAPHQL_VALIDATION_FAILED` under `(unknown)` for invalid queries), but **not** in `http_requests_total`. So
  `graphql_errors_total / http_requests_total` for an operation that also runs over the websocket can go above
  100 %. A subscription counts a code once per event that fails, not once per subscription.
- **Service-specific metrics:**

  | Service | Metric | Type | Labels | Meaning |
  |---|---|---|---|---|
  | rates-service | `rate_ticks_published_total` | counter | `pair` | Rate ticks published (`rate.tick`), by currency pair, e.g. `USD/UAH` |
  | treasury-service | `events_consumed_total` | counter | `type` | Events received through `EventSubscriber`, once per message, e.g. `system.ping` |

- **Host and containers** (collected by Alloy, not by the backends): host CPU, load, memory, disk and network, and
  CPU and memory of every backend container.

`/metrics` listens on its own port (`METRICS_PORT`, default `9464`), never on the app port: it has no public domain,
so `https://staging-wallet-core.wallet.shatori.com/metrics` is a 404. Locally: `curl localhost:9464/metrics`.

## Reading the dashboards

**Dashboards → Internship**:

- **Services** (filters: environment, service): availability, requests per second, error rate %, latency
  p50/p95/p99, the 10 slowest operations, event-loop lag, heap, restarts. Start here when something is slow or
  failing.
- **Server** (filter: environment): CPU, load, memory, disk and network of the host, and CPU and memory per backend
  container.
- **Logs**: what happened in the requests behind a spike.

Going from "slow" to the cause:

1. **Services**: which service and operation? Example: p95 of `mutation convert` jumps from 40 ms to 900 ms at 14:05.
2. **Server**: is it the service or the machine? The wallet-core container at 100 % CPU while the host is fine means
   the service itself.
3. **Logs** at 14:05 for that service: what exactly happened.

How to read the panels:

- **p95 latency** = 95 % of requests were faster than this. Watch p95 and p99, not the average: an average hides
  the slow requests users notice.
- **Error rate** counts server-side failures only (5xx and `INTERNAL_SERVER_ERROR`). A wave of `UNAUTHENTICATED`
  is a client problem; see _GraphQL errors by code_.
- **Heap, event-loop lag and container CPU/memory** are drawn per process, never summed. During a deploy you briefly
  see two lines: the old and the new container.
- **Restarts** counts new processes in the range, deploys included.

## Adding a metric to a feature

Inject `Metrics` and create the metric **once** (constructor or provider factory), then update it where it happens:

```ts
import { Inject, Injectable } from '@nestjs/common';
import type { Counter } from 'prom-client';
import { Metrics } from '../metrics/metrics.js';

@Injectable()
export class ConversionService {
  private readonly conversions: Counter<'from' | 'to'>;

  constructor(@Inject(Metrics) metrics: Metrics) {
    this.conversions = metrics.counter({
      name: 'conversions_total',
      help: 'Booked conversions, by currency pair',
      labelNames: ['from', 'to'],
    });
  }

  async convert(input: ConvertInput) {
    // ...
    this.conversions.inc({ from: input.from, to: input.to });
  }
}
```

- **Counter** (`metrics.counter`): things that happen — requests, events, conversions. Name ends with `_total`.
- **Histogram** (`metrics.histogram`): durations and sizes, to get p95. Name ends with the unit (`_seconds`, `_bytes`).
- **Gauge** (`metrics.gauge`): a current value that goes up and down — open connections, queue length.
- Names: `snake_case`, lowercase, unit at the end (see the [Prometheus naming guide](https://prometheus.io/docs/practices/naming/)).
- A new metric shows up in Grafana **Explore** (Prometheus data source) one scrape after the deploy. A panel for it
  goes into the dashboard JSON in the `ops` repository.

## Label rules

**Every distinct label value creates a new time series**, and Prometheus keeps every series in memory. One label
with an unbounded value can take the whole monitoring down.

- **Allowed**: values from a small, fixed set — environment, service, operation, HTTP status, error code, currency,
  event type.
- **Never**: user ids, request ids, conversion ids, amounts, emails, IP addresses, free text, timestamps. Those belong
  in **logs**, where you search them by `requestId`.
- Ask yourself: "How many different values can this label have in a year?" More than about a hundred → not a label.

## PromQL examples

Use **Explore** with the Prometheus data source. `rate(x[5m])` = per-second rate over the last 5 minutes.

| Question | PromQL |
|---|---|
| Requests per second, per service | `sum by (service) (rate(http_requests_total{env="staging"}[5m]))` |
| Requests per second, per operation | `sum by (operation) (rate(http_requests_total{env="staging", service="wallet-core"}[5m]))` |
| p95 latency of one service | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{env="staging", service="wallet-core"}[5m])))` |
| p95 latency per operation | `histogram_quantile(0.95, sum by (le, operation) (rate(http_request_duration_seconds_bucket{env="staging", service="wallet-core"}[5m])))` |
| GraphQL errors per second, by code | `sum by (code) (rate(graphql_errors_total{env="staging"}[5m]))` |
| Event-loop lag | `nodejs_eventloop_lag_seconds{env="staging"}` |
| Heap used (MB) | `nodejs_heap_size_used_bytes{env="staging"} / 1e6` |

**Error rate %** (server-side failures of one service):

```promql
100 * (
    (sum(rate(http_requests_total{env="staging", service="wallet-core", status=~"5.."}[5m])) or vector(0))
  + (sum(rate(graphql_errors_total{env="staging", service="wallet-core", code="INTERNAL_SERVER_ERROR"}[5m])) or vector(0))
)
/ sum(rate(http_requests_total{env="staging", service="wallet-core"}[5m]))
```

`or vector(0)` keeps the result when a metric has no series yet, for example no 5xx at all.

## Try it

Send some load and watch **Services**: requests per second, latency and the container's CPU go up.

```sh
for i in $(seq 1 200); do
  curl -s https://staging-wallet-core.wallet.shatori.com/graphql \
    -H 'content-type: application/json' -H 'Authorization: Dev 1:wallet' \
    -d '{"query":"query Me { me { id } }"}' > /dev/null
  sleep 0.5
done
```
