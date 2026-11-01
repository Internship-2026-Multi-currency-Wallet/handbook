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
- At least 8 GB of RAM free for Docker when running the full stack.

## 1. Clone the repositories side by side

```sh
mkdir wallet && cd wallet
for repo in ops wallet-core rates-service treasury-service client-web back-office; do
  git clone git@github.com:Internship-2026-Multi-currency-Wallet/$repo.git
done
```

The `ops` stack builds the services from these sibling folders, so keep the names.

## 2. Start the whole system

```sh
cd ops
docker compose up --build
```

The first build takes a few minutes. When `docker compose ps` shows every service as `healthy`, open:

| What | URL |
|---|---|
| Client app (`client-web`) | <http://localhost:8080> |
| Back-office (`back-office`) | <http://localhost:8081> |
| `wallet-core` GraphQL | <http://localhost:3000/graphql> (health: `/health`) |
| `rates-service` GraphQL | <http://localhost:3001/graphql> (subscriptions: `ws://localhost:3001/graphql`) |
| `treasury-service` GraphQL | <http://localhost:3002/graphql> |
| MySQL 8 | `localhost:3306`, user `wallet` / `wallet` (one database per service) |
| Redis | `localhost:6379` |

The backends apply their database migrations on start. Stop with `docker compose down`;
`docker compose down -v` also deletes all data.

## 3. Work on one service

Run the infrastructure in Docker and the service you are changing from source, with hot reload:

```sh
cd ops && docker compose up -d mysql redis       # or the whole stack, then: docker compose stop wallet-core
cd ../wallet-core
npm install
npm run migrate
DEV_AUTH=true npm run start:dev                  # wallet-core :3000, rates-service :3001, treasury-service :3002
```

Frontends: `npm run dev` (client-web on :5173, back-office on :5174). Each repository's README lists its scripts
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

Real login (JWT) replaces this later without changes to your business code — see the backend READMEs (*Seams*).

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
cd ops && CLIENT=kvitka docker compose up -d --build
```

When running a service from source: `CLIENT=kvitka DEV_AUTH=true npm run start:dev`.
The backends report the current client in `health { client }`.

## PayGate

PayGate is not part of the local stack: it is an external sandbox at `https://paygate.wallet.shatori.com`
(docs at `/docs`, available after deploy). API keys come from the mentor — keep them in your local `.env`, never in Git.

## Not there yet

