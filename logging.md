# Logging

All backends write **structured JSON logs**, one line per event. On the deployed environments (`staging`,
`kvitka-uat`, `wisla-uat`) the lines are collected into **Grafana Loki**, together with the logs of the shared Redis
and MySQL and an event every time one of these containers stops; you read and search them in
**Grafana**: <https://logs.wallet.shatori.com> (ask the mentor for an account — you get the *Viewer* role).

Locally (`npm run start:dev`) the same logs are pretty-printed in your terminal.

## A log line

```json
{"level":"info","time":"2026-11-01T17:57:51.921Z","service":"wallet-core","appEnv":"staging","client":"demo",
 "requestId":"trace-123","method":"POST","path":"/graphql","operation":"query Me","status":200,"durationMs":36,
 "msg":"POST /graphql query Me 200 36ms"}
```

| Field | Meaning |
|---|---|
| `level` | `error`, `warn`, `info`, `debug` (`LOG_LEVEL` sets the minimum, default `info`) |
| `service`, `appEnv`, `client` | Which backend, environment and client wrote the line |
| `requestId` | One id per request, GraphQL subscription connection or event — the thread through all services |
| `context` | Class that logged (e.g. `LedgerService`) |
| `msg` + your fields | What happened, with the ids and values needed to understand it |

Every HTTP request gets one **access line** (method, path, GraphQL operation, status, duration — never bodies or
headers). Health probes are logged only at `debug`.

## How to log in code

```ts
import { InjectPinoLogger, PinoLogger } from 'nestjs-pino';

@Injectable()
export class ConversionService {
  constructor(@InjectPinoLogger(ConversionService.name) private readonly logger: PinoLogger) {}

  async convert(userId: string, input: ConvertInput) {
    this.logger.info({ userId, from: input.from, to: input.to, amount: input.amount }, 'conversion requested');
    try {
      // ...
    } catch (err) {
      this.logger.error({ err, userId }, 'conversion failed');
      throw err;
    }
  }
}
```

- **Inject the logger**; no `console.log`, no `new Logger()` from `@nestjs/common`.
- **Fields first, message second.** Put ids and values into fields (`{ conversionId, amount }`), keep the message
  short and constant (`'conversion booked'`) — then you can filter by fields in Grafana.
- **Errors as `{ err }`** — the stack trace is kept.
- **Never log secrets or personal data**: passwords, tokens, API keys, `Authorization`, cookies, card numbers, whole
  request objects or bodies. Known secret field names are redacted automatically (`[REDACTED]`), but do not rely on it.
- **`requestId` is added for you** inside a request, a GraphQL operation and an event handler: events published
  with `EventPublisher` carry it in their meta, and `EventSubscriber` handlers log with the same id.
- Choose the level: `error` — something failed and needs attention; `warn` — unexpected but handled (bad input,
  retries); `info` — business events (user registered, conversion booked); `debug` — details for troubleshooting.

Send your own `x-request-id` header to follow one call. Every response returns its `x-request-id`; for GraphQL
subscriptions put `x-request-id` into the graphql-ws `connectionParams`.

### GraphQL errors (E1-17)

This describes the implementations in
[wallet-core #39](https://github.com/Internship-2026-Multi-currency-Wallet/wallet-core/pull/39),
[rates-service #14](https://github.com/Internship-2026-Multi-currency-Wallet/rates-service/pull/14) and
[treasury-service #10](https://github.com/Internship-2026-Multi-currency-Wallet/treasury-service/pull/10).
Deployment and staging verification are pending.

A failure is **expected** when the original error is a `GraphQLError` with a string `extensions.code`. A code that a
wrapper adds or copies does not count.

| Failure | Log | Response |
|---|---|---|
| Expected | One GraphQL `warn` with `code`, `path` (when available) and `requestId`; no `err`, no extra Nest ERROR | Its message and code |
| Unexpected server failure | `error` with the stack, using the existing redaction rules | Masked: `INTERNAL_SERVER_ERROR` |
| Nest `HttpException` | HTTP: 4xx at `warn`. graphql-ws: 4xx not logged, pending Ihor's decision | Not masked |

- HTTP errors are logged by the Apollo plugin; graphql-ws errors by separate graphql-ws hooks.
- An unexpected resolver failure may produce two ERROR records: one from Nest, one from GraphQL.
- Access lines and separate domain or throttler warnings still appear.
- Client validation errors are not server crashes. Their response and metric codes can differ between services.

## Searching in Grafana

Open **Dashboards → Internship → Logs** (the home page): pick environment, service and level, type a `requestId`
or any text. For ad-hoc queries use **Explore** with the Loki data source.

Labels you can select on: `env` (`staging`, `kvitka-uat`, `wisla-uat`; `shared` for Redis and MySQL), `service`
(`wallet-core`, `rates-service`, `treasury-service`, `redis`, `mysql`), `level`. Everything else is inside the JSON
line (`| json`). Redis and MySQL write plain text, so their lines have **no `level`**: pick *Level = All* to see them.
Loki also shows a `detected_level` it guesses from the text; do not rely on it.

| Question | LogQL |
|---|---|
| Everything one request did, across services | `{env="staging"} \|= "trace-123"` |
| Errors of one service | `{env="staging", service="wallet-core", level="error"}` |
| Errors per service (graph) | `sum by (service) (count_over_time({env="staging", level="error"} [5m]))` |
| Slow requests (> 500 ms) | `{env="staging"} \| json \| durationMs > 500` |
| One GraphQL operation | `{env="staging", service="wallet-core"} \| json \| operation="mutation Convert"` |
| One user's actions | `{env="kvitka-uat"} \| json \| userId="42"` |
| GraphQL errors by code | `sum by (code) (count_over_time({env="staging", level=~"warn\|error"} \| json \| context="GraphQL" [1h]))` |

Logs are kept for **14 days**.

## Why did Redis or MySQL restart?

When a backend logs `ECONNREFUSED`, `ENOTFOUND internship-redis-…` or a MySQL connection error, look at the shared
service itself. Next to its own lines, Loki has one line per **stop of the container**, written by Docker (through the
small `docker-events` service), so it is there even when the process was killed and could not write anything:

```json
{"action":"die","container":"internship-redis-….1.…","exitCode":"137"}
```

| Question | LogQL |
|---|---|
| What Redis said and what happened to it, in time order | `{service="redis"}` |
| Only the stops of MySQL | `{service="mysql"} \|= "\"action\""` |
| Out-of-memory kills of anything we run | `{level="error"} \|= "\"action\":\"oom\""` |
| What the backends saw at that time | `{env="staging", level="error"}` (same time range) |

The same events exist for the backends: a stop of `wallet-core` on `staging` is in `{env="staging", service="wallet-core"}`.

| `action` / `exitCode` | Meaning | `level` |
|---|---|---|
| `die`, `0` | The process stopped by itself, cleanly | `info` |
| `die`, `143` | Stopped with SIGTERM: a normal restart or deploy | `info` |
| `oom`, then `die`, `137` | Killed because it used up its memory limit (Redis has 256 MB on the server) | `error` |
| `die`, `137` without `oom` | Killed (SIGKILL): did not stop in time, or killed by hand | `error` |
| `die`, `1` or another code | Crashed: read its last lines just before the event | `error` |

A normal restart therefore looks like: shutdown lines (Redis: `Received SIGTERM scheduling shutdown…`) → `die` with
`0` or `143` → start-up lines of the new container. An OOM kill has no shutdown lines: the last ordinary line, then
`oom` and `die` `137`, then the start-up lines.

Events are lost while `docker-events` itself is down (for example during a Docker restart on the server).

## Try it

```sh
curl -X POST https://staging-wallet-core.wallet.shatori.com/diagnostics/ping \
  -H 'Authorization: Dev 1:wallet' -H 'x-request-id: my-first-trace'
```

wallet-core publishes a `system.ping` event; every backend of `staging` logs `system.ping received`. Search
`{env="staging"} |= "my-first-trace"` — you see the request in wallet-core and the event in all three services.
