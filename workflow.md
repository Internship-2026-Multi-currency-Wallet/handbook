# Workflow

How we go from an Asana task to code running on a client's environment.

## Tasks in Asana

Project: `https://app.asana.com/1/2427620016900/project/1220109449481948`

- **Sections = epics** (e.g. Identity & access, Ledger & balances, Rates, Top-ups, Conversion,
  Transfers & withdrawals, Compliance & back-office, Treasury & hedging, Delivery & release process, Bonus).
- A **parent task = one feature**: a user story plus feature-level acceptance criteria.
  Client differences (Kvitka / Wisła rules) are part of the feature they belong to — customization lives in every epic.
- **Subtasks** split the work by repository:
  - `[BE] <repo>` — backend work. **Its first acceptance criterion is publishing the GraphQL contract**
    (schema changes in a PR), reviewed by the `[FE]` owner.
  - `[FE] <repo>` — frontend work, built against the contract (with mocks until the backend is ready).
- The parent task is assigned to the **feature lead(s)**. Subtasks may be assigned to different interns.
- Tasks describe **what** and **why** (user story + acceptance criteria). **How** is up to you — design it,
  discuss it, and explain it in the PR.

### Feature lead

The feature lead owns the feature end to end: clarifies questions with the mentor, agrees the contract between
`[BE]` and `[FE]`, keeps the parent task up to date, and checks the acceptance criteria on the environment at the end.
Leads rotate, so everyone leads several features.

### Task lifecycle

1. Pick up / get assigned → read the story and acceptance criteria, ask questions **in the task comments**.
2. `[BE]`: open a contract PR first. `[FE]`: start against mocks right away.
3. Implement in small PRs. Link every PR in the task.
4. When merged and deployed to `staging`, the lead checks the acceptance criteria and marks the task done.

### Status field

Every task and subtask has a **Status** field. Keep it current — it is how the team sees who is doing what.

| Status | When |
|---|---|
| `To do` | Nobody has started yet |
| `In progress` | You picked it up and assigned it to yourself |
| `In review` | Your PR is open — link it in a task comment |
| `Blocked` | You are waiting for another task or a decision — say what in a comment |
| `Done` | The PR is merged and deployed to `staging`; mark the subtask complete as well |

A parent task (feature) is `In progress` while any subtask is in progress, and `Done` only after all subtasks are
done and the lead has shown the demo to the mentor.

## Branches and releases

| Branch | Purpose |
|---|---|
| `feat/<short-name>`, `fix/<short-name>` | Your work. PR into `main` |
| `main` | Integration branch. Auto-deployed to **`staging`** (client `demo`) |
| `version_<X.Y>` | A release cut from `main`. Frozen: only fixes |
| `client/<c>-uat` | A client's environment (e.g. `client/kvitka-uat`). May contain **client-only code** |
| `auto/version_<X.Y>-to-<c>-uat` | Created by the release bot for fan-out PRs |
| `hotfix/<short-name>` | An urgent fix that must reach client branches without waiting for a release |

```
feat/* ──PR──► main ──(cut)──► version_1.3 ──(fan-out bot)──► PR → client/kvitka-uat
                                                     └────────► PR → client/wisla-uat
```

1. **Develop** — feature branches → PR into `main`. `main` deploys to `staging`.
2. **Cut a release** — the mentor runs the `cut-release` workflow, which creates `version_X.Y` from `main`.
3. **Fan-out** — a bot opens one PR per client from `version_X.Y` into `client/<c>-uat`.
   If a client branch contains client-only code, the PR may have **conflicts**. The author of the conflicting
   client-only code resolves them by merging `client/<c>-uat` into the `auto/...` branch.
   Resolving these conflicts well is one of the key skills of this project.
4. **UAT** — a merged fan-out PR deploys to `<c>-uat`; the team checks the release there.
5. **Hotfix** — *process to be defined.* For now, urgent fixes go through the normal flow; ask the mentor
   before fixing anything directly on a client branch.

Several of these workflows (cut-release, fan-out, cherry-pick, branch → environment mapping, onboarding a new client)
are **your tasks** in the delivery epic. The mentor provides only the basic `main` → `staging` deploy.

## Pull requests and review

- Every PR needs **two approvals: one intern + the mentor**. No self-merging without them.
- Keep PRs **small and focused** (aim for < 400 changed lines). Contract PRs and shared-file PRs go separately.
- CI (lint, type check, tests) must be green before review.
- The PR author merges after approvals. Delete the branch after merge.
- Reviewers: be specific and kind. Prefix optional remarks with `nit:`. Approve when it is good enough, not perfect.
- Authors: answer every comment (fix it or explain why not). Do not resolve someone else's comment silently.

### PR description template

```markdown
## What
Short summary of the change.

## Why
Link to the Asana task + one sentence of context.

## How to test
Steps, or which tests cover it. Screenshots / a short video for UI changes.

## Environment variables
None. (Or a table: variable, purpose, value per environment — see "Environment variables" below.)

## Notes
Contract changes, migrations, follow-ups, anything reviewers should look at first.
```

### Environment variables

Deployed backends get their environment variables from **Dokploy**, which only the mentor can access.
So when your change needs a **new** variable, or a **different value** for an existing one, say so in the PR,
and the mentor sets it.

1. **Add it to the code.**
   - Declare it in the service's env schema, so a missing or invalid value stops the service at startup
     with a clear error.
   - Add it to `.env.example` with a working local value (or a placeholder for a secret) and a one-line comment.
2. **Fill in the "Environment variables" section of the PR description**: name, what it is for, and the value
   for each environment. For a secret, write "secret" and where it comes from (e.g. "Resend API key"),
   never the value itself.

   ```markdown
   ## Environment variables
   | Variable | Purpose | staging | kvitka-uat | wisla-uat |
   |---|---|---|---|---|
   | `EMAIL_FROM` | Sender address of system emails | `no-reply@staging.wallet.shatori.com` | `no-reply@kvitka-uat.wallet.shatori.com` | `no-reply@wisla-uat.wallet.shatori.com` |
   | `RESEND_API_KEY` | Resend API key | secret (Resend, staging key) | secret | secret |
   ```

3. **Wait for the mentor** to confirm in the PR that the values are set, **before merging**. Otherwise the
   deploy starts a service with a missing variable and it fails.
   Client environments (`*-uat`) get the change later with the release, but set their values at the same time
   so nothing is forgotten.

The same applies when you **rename or remove** a variable: list it in the PR so the mentor cleans it up.

Frontends have no secrets, and their per-environment values are in Git (`src/config-env.ts` and the deploy
workflow in `ops`), so you change those in your PR as usual. Anything secret must stay in a backend.

### Commit messages

We use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary in imperative mood>
```

| Type | Use for |
|---|---|
| `feat` | New functionality |
| `fix` | Bug fix |
| `refactor` | Code change without behaviour change |
| `test` | Tests only |
| `docs` | Documentation |
| `chore` / `ci` | Tooling, dependencies, CI |

Examples: `feat(ledger): add idempotency key to ledger entries`, `fix(rates): treat silent WebSocket feed as stale`.
PR titles follow the same format.

## Working in parallel

Four people work at the same time, and nobody should wait for someone else's feature to be finished.
**A task may depend on another part's contract, never on its implementation.**

### Contracts first

A contract is an agreed interface:

- GraphQL schema changes (`schema.graphql`),
- a service interface inside a backend (e.g. `LedgerService`),
- an event payload (e.g. `conversion.completed`).

Contracts are published as a PR, together with a **fake** implementation, and **reviewed by the consumers**.
Changing a contract later = a new PR, again reviewed by the consumers.

### Fakes and mocks

The skeleton repositories come with seams that let you build before the real parts exist:

| Seam | Lets you… |
|---|---|
| Dev auth mode (`Authorization: Dev <userId>:<permissions>`) + `@CurrentUser()` | Write business logic before real JWT login exists |
| Permission decorator / component, permissive in dev mode | Build back-office pages before roles exist |
| Settings stub with defaults | Use spread, fees and limits before the settings store exists |
| Fake rates client | Build conversions before `rates-service` is live |
| Typed event catalog + publisher/subscriber helpers | Producers and consumers agree on the same event shapes |
| Generated GraphQL types + Mock Service Worker handlers (`VITE_USE_MOCKS=true`) | Build every screen before its backend exists |

Exact names may differ slightly in the skeletons — check each repo's README.

### Contract day and integration day

| When | What happens |
|---|---|
| **Contract day** (start of a block of work, first half-day) | Everyone writes and opens contract PRs + fakes. Consumers review. This is the main cross-team conversation |
| **Parallel work** | Everyone builds against contracts and fakes |
| **Integration day** (end of the block) | Fakes are switched off, real implementations wired together, end-to-end check on `staging`, demo to the mentor |

### Avoiding merge conflicts

- **Backend**: one NestJS module per feature (`auth/`, `ledger/`, `conversion/`, …). Touch only your modules.
- **Frontend**: one folder per feature (`features/<name>`) with its own Redux slice and routes.
- **Shared files** (`app.module.ts`, root schema, router, navigation) change only in **small, dedicated PRs**. Merge them quickly.
- **Migrations**: timestamp-named Knex migrations. **Never edit a merged migration** — add a new one.
- Rebase or merge `main` into your branch often (at least daily).
- If you and someone else must touch the same file, talk first and agree who goes first.
