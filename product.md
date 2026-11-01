# Product

## The wallet in one paragraph

A user of one of our clients (say, Kvitka Bank) signs up, passes KYC, tops up their wallet with a card,
and now holds money in one or more currencies. They can convert between currencies at a **live rate**,
send money to another user of the same client by email or phone, and withdraw to a card. Client staff
operate the platform through the **back-office**. Behind the scenes, the platform aggregates exchange rates
from public sources and tracks and hedges its own currency **exposure**.

## Currencies

| Currency | Type | Notes |
|---|---|---|
| UAH | Fiat | Main currency for Kvitka Bank |
- USD | Fiat
- EUR | Fiat
| PLN | Fiat | Main currency for Wisła Transfer |
| USDT | Stablecoin | Tracks USD; important for Stablo later |

Amounts are stored as **integers in minor units** (e.g. `10050` = 100.50). Never use floating-point numbers for money.

## MVP features

### Client app (`client-web`) — for end users

| # | Feature | Notes |
|---|---|---|
| 1 | Registration & login | Email + password; optional **Sign in with Google** (a logged-in user links their Google account first — there is no sign-up via Google) |
| 2 | KYC submission | Personal data + document upload; reviewed **manually** by staff |
| 3 | Card top-up | Through PayGate (redirect to a hosted payment page + webhook) |
| 4 | Balances | One balance per currency |
| 5 | Conversion | Get a **quote** at a live rate → confirm before it expires |
| 6 | Transfer | To another user by email or phone |
| 7 | Withdrawal to card | A request; approved manually in the back-office |
| 8 | Transaction history | Filters, pagination |
| 9 | Conditional conversion | "Convert when the rate reaches X" |

### Back-office (`back-office`) — for client staff and platform admins

| # | Feature | Notes |
|---|---|---|
| 1 | Users | List, details, **manual KYC review** (approve / reject with reason) |
| 2 | Transactions | List with filters |
| 3 | Manual balance adjustment | With **maker/checker**: one person creates, another approves |
| 4 | Client settings | Spread, fees, limits, hedging threshold, enabled features |
| 5 | Rates monitor | State of each rate source; enable/disable a source |
| 6 | Audit log | Who did what and when |
| 7 | Withdrawal approvals | Approve (mark paid) or reject (release the held funds) |
| 8 | Exposure & hedging dashboard | Live exposure per currency, hedge history, "hedge now" |

### Platform internals

- **Rates aggregator** — collects rates from public sources, drops stale and outlier values, streams a consensus rate.
- **Ledger** — the source of truth for balances; every operation is idempotent.
- **Treasury & hedging** — tracks exposure and hedges it when it crosses a threshold ("paper" execution, no real trades).
- **Customization mechanism** — lets one codebase behave differently per client. See [customizations.md](customizations.md).

### Not in the MVP

Mobile app · automated KYC · a real liquidity provider. A **bonus** epic (Web3 on testnets, AWS staging) may come later.

## Clients

Each deployment serves exactly **one** client. The differences below are intentional: each one is implemented
with a different customization technique, so you will get to practise all of them.

### Kvitka Bank (`kvitka`) — Ukrainian neobank

- KYC is mandatory before the first top-up
- Phone number is required at sign-up
- 0.5% cashback on every conversion
- Conversions only on weekdays, 09:00–18:00 Kyiv time

### Wisła Transfer (`wisla`) — Polish money-transfer service

- KYC is required only for transfers above 1,000 PLN
- Fee = fixed amount + percentage
- "Split the bill" between several users — exists only for Wisła
- Phone number is required only before the first transfer
- Sign in with Google is disabled

### Stablo (`stablo`) — crypto-friendly wallet *(later)*

USDT focus, lower limits, one exclusive feature. Onboarding Stablo (config, branch, environment, domain)
is itself a team task — a good test of how easy our platform is to extend.

### `demo` — default client

Runs on `staging`. Has no customizations: it shows the "core" product exactly as built on `main`.

## Why the platform needs hedging

User balances are the platform's **debts** to its users. Suppose the rate is 1.11 USD per EUR and, during a day,
users convert 100,000 USD into 90,090 EUR. The platform now owes 90,090 EUR more, but it still physically holds
the 100,000 USD it received. If EUR rises to 1.13 before the platform buys euros, buying 90,090 EUR costs
101,802 USD — a **loss of 1,802 USD**, possibly more than the spread it earned.

**Hedging** fixes this: buy the currency you owe, sell the one you received. We use **threshold hedging**:
the treasury service accumulates net exposure per currency and hedges in one trade once it exceeds the client's
threshold (e.g. 10,000 USD equivalent). Trades are executed on paper at the current consensus rate.

## Glossary

| Term | Meaning |
|---|---|
| **White-label** | One product, sold to several companies, each running it under its own brand and rules |
| **Client** | A company that buys our wallet (Kvitka, Wisła, …). Not to be confused with *user* |
| **User** | An end user of a client's wallet |
| **Staff** | A client's employee using the back-office. Has **permissions** like `bo:users` (read) or `bo:users:modify` |
| **Ledger** | The record of all money movements. Balances are derived from ledger entries, never edited directly |
| **Double-entry** | Every movement is recorded as a debit on one account and a credit on another, so totals always balance |
| **Hold** | Funds reserved (e.g. for a pending withdrawal) — not spendable, not yet gone |
| **Minor units** | The smallest unit of a currency (cents, kopiykas). Amounts are stored as integers in minor units |
| **Rate source** | An external public API providing exchange rates (central banks, banks, crypto exchanges) |
| **Tick** | One rate update from a source |
| **Mid rate** | The middle between buy (bid) and sell (ask) prices |
| **Consensus rate** | Our aggregated rate across sources, after removing stale and outlier values |
| **Stale** | A rate that has not been updated for too long to be trusted |
| **Outlier** | A rate that differs too much from the others (probably wrong) |
| **Triangulation** | Deriving a rate via a third currency, e.g. EUR/USD from EUR/USDT and USDT/USD |
| **Spread** | The margin a client adds on top of the market rate. It is how the client earns on conversions |
| **Fee** | An explicit charge for an operation (e.g. fixed + percentage on a transfer) |
| **Quote** | A rate offered to a user for a specific conversion, valid for a short time (TTL). The user must confirm before it expires |
| **Conditional conversion** | An order that executes automatically when the rate reaches a target |
| **Exposure** | The platform's net position per currency: what it owes users vs. what it actually holds. Exposure = FX risk |
| **Hedging** | Buying the currency you owe and selling the one you received, so exchange-rate moves cannot hurt you |
| **Threshold hedging** | Hedge only when exposure exceeds a per-client threshold, in one trade, instead of after every conversion |
| **Paper execution** | Simulated trade at the current market rate — recorded, but no real money moves |
| **P&L** | Profit and loss: spread earned minus losses from rate moves |
| **KYC** | "Know Your Customer" — verifying a user's identity. Ours is a manual review in the back-office |
| **Maker/checker** | Four-eyes rule: one staff member creates a sensitive action, a *different* one approves it |
| **Audit log** | An append-only record of staff actions |
| **Idempotency** | Repeating the same request gives the same result and no duplicate side effects (e.g. no double crediting) |
| **Idempotency key** | A unique key sent with a request so the server can recognise a retry |
| **Webhook** | An HTTP call a provider (PayGate) makes to us when something happens, e.g. a payment succeeded |
| **HMAC signature** | A hash proving a webhook really came from the provider and was not changed |
| **UAT** | User Acceptance Testing — the environment where a client's version is checked |
| **Contract** | An agreed interface between two parts: GraphQL schema, service interface, or event payload |
| **Fake / mock** | A simple stand-in implementation of a contract, used until the real one exists |
| **Feature lead** | The intern who drives a feature end to end and coordinates its subtasks |
