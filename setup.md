# Local setup

> **Placeholder.** Step-by-step instructions for running the whole system locally with **docker-compose**
> (MySQL, Redis and all services) will be added here once the skeleton repositories are ready.

## Prerequisites

Install these before day one:

| Tool | Version | Check |
|---|---|---|
| **Node.js** | Current LTS | `node -v` |
| **Docker** (Docker Desktop or Docker Engine + Compose v2) | Recent | `docker compose version` |
| **Git** | Recent | `git --version` |
| **GitHub account** | — | Send your username to the mentor to get access to the organization |

Also recommended:

- An editor with TypeScript, ESLint and Prettier support (e.g. VS Code).
- SSH key or a credential helper configured for GitHub.
- At least 8 GB of RAM free for Docker when running the full stack.

## What to expect

- One command (from the `ops` repository) to start MySQL, Redis and all services.
- Seed data with demo clients, users, staff and balances.
- A `CLIENT` environment variable to choose which client you run locally (`demo`, `kvitka`, `wisla`).
- Frontends that can run against mocks (`VITE_USE_MOCKS=true`) without any backend.
