# Local setup

How to run the whole system on your machine. The `ops` repository README has the same steps in more detail.

## Prerequisites

| Tool | Version | Check |
|---|---|---|
| **Node.js** | 26 (current LTS). Each repository has an `.nvmrc`, so `nvm use` picks it | `node -v` |
| **Docker** (Docker Desktop or Docker Engine + Compose v2) | Recent | `docker compose version` |
| **Git** | Recent | `git --version` |
| **GitHub account** | — | Send your username to the mentor to get access to the organization |

Also recommended:

- An editor with TypeScript, ESLint and Prettier support (e.g. VS Code).
- SSH key or a credential helper configured for GitHub.
- Docker memory: the infrastructure (MySQL + Redis) needs well under 1 GiB. Building images needs more —
  parallel builds crash a Docker VM with 2 GiB, so `dev.sh` builds one image at a time. To run the whole stack,
  give Docker at least 4 GiB (Docker Desktop: *Settings → Resources*; Colima: `colima start --memory 4`).

## 1. Clone the repositories side by side

```sh
mkdir wallet && cd wallet
for repo in ops wallet-core rates-service treasury-service client-web back-office; do
  git clone git@github.com:Internship-2026-Multi-currency-Wallet/$repo.git
done
```

The `ops` stack builds the services from these sibling folders, and every repository calls `../ops/dev.sh`, so keep
the names.

## 2. Start the infrastructure — and only what else you need

You usually change one service and run it from source with hot reload. It needs MySQL and Redis, sometimes one or
two other services. `ops/dev.sh` starts exactly that: the infrastructure is always on, every application service is
opt-in.

```sh
cd ops
./dev.sh up                                   # MySQL (all databases) + Redis — no application services
./dev.sh up rates-service treasury-service    # + these services in containers
./dev.sh up all                               # the whole system
./dev.sh ps                                   # also: logs <service>, build <service>, info, help
./dev.sh down                                 # stop (data is kept)
```

Every application repository has the same commands as npm scripts, so you never leave your service folder:

```sh
npm run infra:up                              # ../ops/dev.sh up
npm run infra -- up rates-service             # ../ops/dev.sh up rates-service
npm run infra:down                            # ../ops/dev.sh down
```

`dev.sh up` waits until everything is `healthy` and prints the ports and the env values a service run from source
should use (`DATABASE_URL`, `REDIS_URL`, URLs of the other services). Each service has the same port in Docker and
from source:

| What | URL |
|---|---|
| Client app (`client-web`) | <http://localhost:8080> in Docker, <http://localhost:5173> with `npm run dev` |
| Back-office (`back-office`) | <http://localhost:8081> in Docker, <http://localhost:5174> with `npm run dev` |
| `wallet-core` GraphQL | <http://localhost:3000/graphql> (health: `/health`) |
| `rates-service` GraphQL | <http://localhost:3001/graphql> (subscriptions: `ws://localhost:3001/graphql`) |
| `treasury-service` GraphQL | <http://localhost:3002/graphql> |
| MySQL 8 | `localhost:3306`, user `wallet` / `wallet` (one database per service) |
| Redis | `localhost:6379` |

The backends apply their database migrations on start. `./dev.sh down` stops everything;
`./dev.sh down --volumes` also deletes all data.

## 3. Work on one service

```sh
cd wallet-core
npm run infra:up                              # or: npm run infra -- up rates-service  (+ a neighbour in Docker)
npm install
npm run migrate
DEV_AUTH=true npm run start:dev               # wallet-core :3000, rates-service :3001, treasury-service :3002
```

- Your service reaches MySQL, Redis and the containers at `localhost:<port>`; the defaults in `.env.example`
  already match.
- Containers reach your service at `host.docker.internal:<port>` (preconfigured as `WALLET_CORE_URL`,
  `RATES_SERVICE_URL`, `TREASURY_SERVICE_URL`).
- Do not run the same service in Docker and from source at once (same port): `./dev.sh down wallet-core` first.

Frontends: `npm run dev` (client-web on :5173, back-office on :5174), with the backends started by
`npm run infra -- up wallet-core rates-service` or run from source. Each repository's README lists its scripts
(lint, tests, e2e tests, codegen, migrations).

## Dev auth: calling the API before real login exists

Locally the backends run with `DEV_AUTH=true`. Every request can then say who it is:

```
Authorization: Dev <userId>:<perm1,perm2>
```

```sh
curl -s localhost:3000/graphql -H 'content-type: application/json' \
  -H 'Authorization: Dev 1:wallet' -d '{"query":"{ me { id permissions } }"}'
```

- `Dev 1:wallet` — an end user; `Dev 100:bo:users,bo:users:modify` — a staff member with two permissions.
- Without the header, protected queries return `UNAUTHENTICATED` (HTTP endpoints: 401).
- GraphQL subscriptions: send the same value as `Authorization` in the graphql-ws connection params.
- Frontends send the dev user from `VITE_DEV_USER` (e.g. `VITE_DEV_USER=1:wallet` in `.env.local`).

Dev auth is allowed only when `APP_ENV` is `local` or `staging`; in client UAT environments the services refuse to
start with `DEV_AUTH=true`. Real login (JWT) replaces it later without changes to your business code — see the
backend READMEs (*Seams*).

`APP_ENV` (default `local`) also prefixes every Redis channel, stream and queue (`local:rate.tick`), so several
environments can share one Redis. Use the event helpers in each backend instead of raw Redis names.

## Frontends without backends: mocks

```sh
cd client-web    # or back-office
npm run dev:mocks
```

`VITE_USE_MOCKS=true` makes Mock Service Worker answer every GraphQL call, so you can build any screen before its
backend exists. Types come from each backend's `schema.graphql`: `npm run schema:pull && npm run codegen`.

## Choosing the client

Each deployment serves one client, set by `CLIENT` (`demo`, `kvitka`, `wisla`; default `demo`):

```sh
cd ops && CLIENT=kvitka ./dev.sh up all
```

When running a service from source: `CLIENT=kvitka DEV_AUTH=true npm run start:dev`.
The backends report the current client in `health { client }`.

## Deployed environments

Besides your local stack there are three shared environments, deployed automatically from Git:

| Environment | Branch | Client app | Back-office | APIs |
|---|---|---|---|---|
| `staging` | `main` | <https://staging.wallet.shatori.com> | <https://staging-admin.wallet.shatori.com> | `https://staging-{wallet-core,rates,treasury}.wallet.shatori.com` |
| `kvitka-uat` | `client/kvitka-uat` | <https://kvitka-uat.wallet.shatori.com> | <https://kvitka-uat-admin.wallet.shatori.com> | `https://kvitka-uat-{wallet-core,rates,treasury}.wallet.shatori.com` |
| `wisla-uat` | `client/wisla-uat` | <https://wisla-uat.wallet.shatori.com> | <https://wisla-uat-admin.wallet.shatori.com> | `https://wisla-uat-{wallet-core,rates,treasury}.wallet.shatori.com` |

Dev auth (`Authorization: Dev 1:wallet`) works on `staging` only. Check a backend with
`curl https://staging-wallet-core.wallet.shatori.com/health`. Details: [architecture.md](architecture.md#environments).

## PayGate

PayGate is not part of the local stack: it is an external sandbox at <https://paygate.wallet.shatori.com>
(docs at <https://paygate.wallet.shatori.com/docs>). API keys come from the mentor — keep them in your local `.env`, never in Git.

## Not there yet

