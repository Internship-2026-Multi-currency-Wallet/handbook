# Client customizations

One codebase serves several clients. Kvitka and Wisła want different rules, and Stablo will come later.
This page explains **how** we let behaviour differ per client without turning the code into a pile of
`if (client === 'kvitka')`.

> The customization module itself is **built by you** (it is a task). The snippets below are illustrative:
> they show the idea and the intended shape of the API, not a finished implementation.

## Which client am I?

Each deployment serves **one client**, set by the `CLIENT` environment variable:

| Environment | `CLIENT` |
|---|---|
| `staging` | `demo` (no customizations) |
| `kvitka-uat` | `kvitka` |
| `wisla-uat` | `wisla` |

Locally, you choose the client by setting `CLIENT` before starting the services. Running the same tests with
`CLIENT=kvitka` and `CLIENT=wisla` should show different behaviour where the clients differ.

## The customization ladder

Always use the **lowest rung that works**. Each rung up is more powerful and more expensive to maintain.

| # | Rung | Use when… | Cost |
|---|---|---|---|
| 1 | **Setting** | The difference is a value or an on/off switch | Lowest: change data, not code |
| 2 | **Per-client config** | The difference is structured data (a rule with several parameters) | Low: a JSON object, validated and typed |
| 3 | **Hook** | Extra or different *logic* before/after a core operation | Medium: code in the client's module, core stays untouched |
| 4 | **Client-only endpoint** | A whole feature exists for one client only | Medium: guarded code in shared repo, invisible to others |
| 5 | **Code on the client branch** | The change is deeply client-specific and does not belong in the shared product | Highest: must be carried through every release (fan-out conflicts) |

How our clients' differences map onto the ladder:

| Client rule | Rung |
|---|---|
| Kvitka: KYC before the first top-up | 1 · Setting |
| Kvitka: phone required at sign-up · Wisła: phone required before first transfer | 1 · Setting |
| Wisła: Sign in with Google disabled | 1 · Setting |
| Wisła: KYC only for transfers above 1,000 PLN | 2 · Per-client config |
| Kvitka: 0.5% cashback on conversions | 3 · Hook |
| Wisła: fee = fixed + percentage | 3 · Hook |
| Wisła: "Split the bill" | 4 · Client-only endpoint |
| Kvitka: conversions only on weekdays 09:00–18:00 Kyiv time | 5 · Code on `client/kvitka-uat` |

## Backend (`wallet-core`)

### 1. Settings

Typed values with safe defaults, editable per client in the back-office (spread, fees, limits, feature switches).

How they are stored:

- **Defaults live in the platform code.** Every setting has one, so a fresh environment works with an empty database.
- **The database stores only overrides.** A row appears only when a client changes a value; delete the row and the default applies again.
- **One environment = one client = one database.** On `kvitka-uat` the database holds only Kvitka's overrides; there is no table of "all clients".

```ts
const required = await settings.get('kyc.requiredBeforeTopUp'); // false by default, true for Kvitka
if (required && user.kycStatus !== 'approved') throw new KycRequiredError();
```

Some settings are **public**: the frontend reads them to decide what to show (see [Frontend](#frontend)).

### 2. Per-client config

For rules that need several parameters. Validate the shape on startup — a broken config should fail fast.

```json
{ "kycThreshold": { "currency": "PLN", "amount": 100000 } }
```

(Amounts in minor units: `100000` = 1,000.00 PLN.)

### 3. Hooks

Core code marks an extension point. Client code attaches `before` / `after` handlers to it.
Core modules **never import** client code.

```ts
// core: conversion/conversion.service.ts
@Customizable()
async execute(input: ConvertInput): Promise<Conversion> { /* ... */ }

// client: customizations/clients/kvitka/conversion.customization.ts
@ClientHook('kvitka', ConversionService, 'execute', 'after')
async addCashback({ result }) { /* credit 0.5% cashback */ }
```

- `before` handlers can change the arguments; `after` handlers can change the result.
- Handlers of another client never run.
- Hook targets should be type-checked: a typo in the class or method name is a compile error.
- Decide explicitly whether a failing handler aborts the operation (validations) or is only logged (side effects).

### 4. Client-only endpoints

A guard on a resolver makes it exist for one client only. For everyone else it behaves as if it did not exist.

```ts
@ClientFeature('wisla')
@Mutation(() => BillSplit)
splitBill(@Args('input') input: SplitBillInput) { /* ... */ }
```

The inverse — hiding a core endpoint for some clients — works the same way (e.g. `@ClientDisabled('wisla')`).

### 5. Code on the client branch

The last resort. The code lives only on `client/<c>-uat` and never reaches `main`.
Every release fan-out has to carry it forward, which is where merge conflicts come from. Keep it small and isolated
(new files rather than edits in core files) so conflicts stay rare.

## Frontend (`client-web`, `back-office`)

Three tools, used in this order:

### 1. Product flags from client settings (runtime)

The backend exposes the client's **public settings**, including which products/pages are enabled.
Routes and navigation are built from those flags by **one pure function** — a disabled product has no route
and no menu item.

```ts
const routes = buildRoutes(publicSettings.features);
// { splitBill: false } → no /split-bill route, no "Split the bill" menu item
```

Because it is a pure function, it is easy to test with different flag sets.

### 2. Build-time config with env overrides

One config file (e.g. `config-env.ts`) holds per-environment values (service URLs, build flags).
Each value can be overridden by a `VITE_*` variable and has a **safe default**.

```ts
export const config = {
  walletCoreUrl: import.meta.env.VITE_WALLET_CORE_URL ?? 'http://localhost:3000/graphql',
  useMocks: import.meta.env.VITE_USE_MOCKS === 'true',
};
```

### 3. Client-only UI on the client branch

UI that only one client has (and that is not just a product flag) lives as code on that client's branch.
Never write `if (client === 'kvitka')` in shared frontend code.

### Branding

Branding works through **brand files on the client branch**. Shared code always refers to the same
few files; each branch has its own version of them.

| What | Where |
|---|---|
| Logo, favicon | fixed paths under `public/brand/` |
| Product name | `siteName` in `config-env.ts` |
| Colours, font | the brand theme file (MUI palette and typography) |
| Page title, web manifest | `index.html`, `manifest.json` |

- `main` contains the **Demo Wallet** brand (seen on `staging`).
- `client/kvitka-uat` and `client/wisla-uat` contain the same files with Kvitka and Wisła versions.
- Components never hard-code colours or names; they use the theme and `siteName`.
- When `main` is merged into a client branch, brand files normally don't conflict, because `main` rarely touches them.
  If it does, resolve the conflict in favour of the client's version.
- Assets and palettes: [`brands/`](brands/README.md).

To see a client's brand locally, check out its branch.

## Checklist before adding a customization

- [ ] Can this be a setting? Then make it a setting.
- [ ] Does core code stay free of client names?
- [ ] Is the default behaviour (`demo`) unchanged?
- [ ] Are there tests that run with different `CLIENT` values?
- [ ] If it lives on a client branch: is it isolated in new files, and does the team know it exists?
