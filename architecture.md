# Architecture

## Big picture

```mermaid
flowchart LR
    subgraph Frontends
        CW[client-web<br/>React]
        BO[back-office<br/>React]
    end

    subgraph Backend services
        WC[wallet-core<br/>NestJS]
        RS[rates-service<br/>NestJS]
        TS[treasury-service<br/>NestJS]
    end

    PG[PayGate<br/>sandbox payment gateway]
    SRC[Public rate sources<br/>central banks, banks,<br/>crypto exchanges]
    R[(Redis<br/>pub/sub + Streams + BullMQ)]
    DB[(MySQL<br/>one database per service)]

    CW -- GraphQL --> WC
    CW -- GraphQL --> RS
    BO -- GraphQL --> WC
    BO -- GraphQL --> RS
    BO -- GraphQL --> TS

    WC -- GraphQL --> RS
    WC -- create payment --> PG
    PG -- webhook --> WC

    SRC -- REST poll / WebSocket --> RS
    RS -- rate ticks: pub/sub --> R
    WC -- conversion.completed: Stream --> R
    R --> TS
    R --> WC

    WC --- DB
    RS --- DB
    TS --- DB
```

In words: the two frontends talk to the backend services over **GraphQL**. `rates-service` pulls rates from public
sources and publishes live ticks. `wallet-core` owns users, money and client settings, and talks to PayGate for
card top-ups. `treasury-service` listens to completed conversions and hedges the resulting exposure.

## Repositories

| Repo | What it is | Main responsibilities |
|---|---|---|
| `wallet-core` | NestJS service | Auth & JWT, users, staff & permissions, client settings, customization module, KYC (manual), ledger & balances, conversions & quotes, transfers, withdrawals, top-ups via PayGate, audit log |
| `rates-service` | NestJS service | Source adapters (polling + WebSocket), aggregation (staleness, outliers, consensus, triangulation), live rate stream, source monitor |
| `treasury-service` | NestJS service, gateway + worker processes | Exposure per currency, threshold hedging worker (paper execution), hedge history, P&L |
| `client-web` | React app | End-user wallet |
| `back-office` | React app | Staff & platform admin UI |
| `ops` | Infrastructure | Shared CI/CD workflows, release automation, docker-compose for the whole system, list of clients and environments |
| `paygate` | **External** sandbox provider | Fake card payment gateway. Maintained by the mentor — treat it exactly like a real third-party API: you only use its public API and docs |

You get **skeleton repositories** (empty NestJS / React apps with tooling configured). The features are yours to build.

## Tech stack

| Area | Choice |
|---|---|
| Language | TypeScript everywhere |
| Backend framework | **NestJS** + **GraphQL** (Apollo) |
| Database | **MySQL**, one database per service |
| DB access | **Knex** (query builder + migrations) |
| Background jobs | **BullMQ** (on Redis) |
| Events | **Redis** pub/sub and Redis Streams |
| Frontend | **React** + **Redux Toolkit** + **MUI** (incl. DataGrid) |
| Frontend GraphQL | `graphql-request` (queries/mutations), `graphql-ws` (subscriptions), types generated with graphql-codegen |
| Frontend build | **Vite** |
| Tests | **Jest** (backend), **Vitest** + **Playwright** (frontend) |
| CI/CD | GitHub Actions → container images → deploy per environment |

Both frontends use the same stack, so you can move between them easily.

## How services communicate

| Kind | Mechanism | Use for | Example |
|---|---|---|---|
| Frontend → backend | GraphQL queries & mutations | Everything the UI reads or changes | `balances`, `createQuote`, `confirmConversion` |
| Frontend ← backend, live | GraphQL subscriptions (WebSocket) | Live data | `rateUpdated`, balance changes, exposure |
| Service → service, sync | GraphQL | Asking another service now, needing an answer | `wallet-core` asks `rates-service` for the current rate |
| Service → service, live data | **Redis pub/sub** | Fire-and-forget; missing one message is fine | `rate.tick` |
| Service → service, events | **Redis Streams** + consumer groups | Events that must **not** be lost; consumers ack and retry | `conversion.completed` → `treasury-service` |

Rules of thumb:

- If losing the message would lose money or break reporting → **Stream**, not pub/sub.
- Consumers must be **idempotent**: the same event may arrive twice.
- Event payloads are contracts: they are typed, versioned with care, and reviewed by consumers.

## Authentication & authorization

- `wallet-core` is the only service that **issues JWTs** (access + refresh) for users and staff.
- Other services **verify the signature** with the public key — they never call `wallet-core` to check a token.
- Login methods: email + password; Sign in with Google for users who have linked their Google account (can be disabled per client).
- Staff actions are protected by **permission strings**, e.g. `bo:users` (view) and `bo:users:modify` (change).
  The back-office hides pages and buttons the staff member has no permission for; the backend enforces it regardless.
- Until real auth exists, the skeletons provide a **dev mode** so you can build business logic without real tokens
  (see [workflow.md](workflow.md#working-in-parallel)).

## External rate sources

All sources are **free public APIs** that work without an API key. Check each source's terms of use
(rate limits, attribution, caching) before relying on it.

| Source | Type | Speed | Good for learning |
|---|---|---|---|
| National Bank of Ukraine (NBU) | REST poll | Daily official rate | UAH reference rate, business days |
| PrivatBank | REST poll | Intraday | Buy/sell rates |
| Monobank | REST poll | Intraday, strict rate limit | Caching, respecting rate limits |
| Frankfurter (ECB data) | REST poll | Daily, working days | EUR-based rates, weekend gaps |
| National Bank of Poland (NBP) | REST poll | Daily, working days | PLN reference rate |
| ExchangeRate-API (open access) | REST poll | Daily | Attribution requirement |
| Kraken | WebSocket stream | Real-time, 24/7 | Reconnects, heartbeats, silent-feed detection |
| Binance | WebSocket stream | Real-time | Triangulation via USDT |
| Coinbase Exchange | WebSocket stream | Real-time | Another stablecoin/fiat source |

Mixing slow official sources with real-time exchange feeds is deliberate: it creates real aggregation problems —
different update speeds, bid/ask vs. mid, staleness per source, outliers.

## Environments

Each environment serves **exactly one client**, chosen by the `CLIENT` environment variable, and has its own databases.
There is no client production environment — UAT is the final stage in this project.

| Environment | Deployed from branch | `CLIENT` | Client app URL |
|---|---|---|---|
| `staging` | `main` | `demo` | `https://staging.wallet.shatori.com` |
| `kvitka-uat` | `client/kvitka-uat` | `kvitka` | `https://kvitka-uat.wallet.shatori.com` |
| `wisla-uat` | `client/wisla-uat` | `wisla` | `https://wisla-uat.wallet.shatori.com` |
| `stablo-uat` *(later)* | `client/stablo-uat` | `stablo` | `https://stablo-uat.wallet.shatori.com` |

- Sub-domains for the back-office and APIs follow the same pattern (e.g. `admin.<env>.wallet.shatori.com`);
  the exact names will be confirmed when the environments are deployed.
- PayGate is **one shared instance**: `https://paygate.wallet.shatori.com` (docs at `/docs`, available after deploy).
- Deploys happen **only through CI**: a push to `main` or `client/**` builds images and redeploys the matching environment.
  Nobody deploys by hand. See [workflow.md](workflow.md#branches-and-releases).
- Locally, you run the whole system with docker-compose — see [setup.md](setup.md).
