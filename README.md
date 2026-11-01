# Wallet Internship Handbook

Welcome! This handbook is everything you need to start working on the internship project.
It is a living document: if something is unclear or out of date, ask in the team chat or open a PR.

> Status: **draft**. Some details (local setup, exact sub-domains) will be filled in once the skeleton repositories are ready.

## What we are building

A **white-label multi-currency wallet**. "White-label" means we build one product and sell it to several
fintech companies (our *clients*). Each client runs the wallet under its own brand and with its own rules,
for its own end users.

End users can top up by card, hold balances in **UAH, USD, EUR, PLN and USDT**, convert between currencies
at a live rate, send money to other users and withdraw to a card. Client staff use a **back-office** to review
KYC, approve withdrawals, manage settings and watch the platform's currency exposure.

No real money is involved anywhere. Card payments go through **PayGate**, a sandbox payment gateway that
behaves like a real third-party provider.

## Our clients (fictional)

| Client | Code | In short |
|---|---|---|
| **Kvitka Bank** | `kvitka` | Ukrainian neobank: UAH-centric, strict KYC, cashback on conversions |
| **Wisła Transfer** | `wisla` | Polish money-transfer service: PLN & EUR, own fee model, "split the bill" |
| **Stablo** | `stablo` | Crypto-friendly wallet, USDT-focused. **Added later** as a team task |

Plus `demo` — the default client with no customizations, used on `staging`.
Details are in [product.md](product.md).

## How you will work

- **Team of four**, no fixed areas. Each feature has a **feature lead**; leads rotate so everyone sees
  every part of the system (backend, frontend, services, release process).
- Several repositories (backend services, two frontends, ops). See [architecture.md](architecture.md).
- Work happens in parallel: teams agree on **contracts** first and build against **fakes**, so nobody waits.
  See [workflow.md](workflow.md).

## Reading order

1. **This README**
2. [product.md](product.md) — what the product does, the clients, glossary
3. [architecture.md](architecture.md) — repositories, tech stack, how services talk, environments
4. [workflow.md](workflow.md) — branches, releases, PRs, Asana, working in parallel
5. [customizations.md](customizations.md) — how one codebase serves different clients
6. [setup.md](setup.md) — getting the system running locally

Plan about an hour for the first read. You do not need to remember everything — come back when a task touches a topic.

## Where things live

| What | Where |
|---|---|
| Tasks | Asana project: `https://app.asana.com/1/2427620016900/project/1220109449481948` |
| Code | GitHub organization (you will receive an invitation) |
| This handbook | The `handbook` repository |
| PayGate integration docs | <https://paygate.wallet.shatori.com/docs> *(available after deploy)* |
| Environments | `*.wallet.shatori.com` — see [architecture.md](architecture.md#environments) |

## Ground rules

- **English only** in code, commits, PRs, Asana and docs.
- Every PR is reviewed by **one other intern and the mentor** before merging.
- No real personal data, card numbers or secrets in code, tickets or screenshots.
- Ask early. Being blocked for more than ~30 minutes is a signal to ask, not to push harder.
