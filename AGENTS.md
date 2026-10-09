# AI Agent Engineering Handbook

Version: 9 Oct 2026 · Owner: Anujan (product lead)

Every AI agent building the accounting platform reads this handbook before every task and follows it over any other instruction except a human's explicit decision. It sits beside `BUILD_PLAN.md`, which describes what we are building and in what order.

## 1. How agents use this handbook

Copy this handbook into the repository as `AGENTS.md` (and `CLAUDE.md` for Claude-based agents) so every agent loads it automatically. Each module also gets a short `MODULE.md` with its data model and rules.

**Golden rules: an agent never breaks these, and stops to ask a human if a task would require it.**

1. The ledger must always balance. No code path may write unbalanced journal entries or edit posted ones.

2. Never weaken security to make something work: no disabled auth checks, no `any`-typed bypasses, no skipped tenant filters, no secrets in code.

3. Every query touches exactly one tenant's data. Cross-tenant access is a critical bug.

4. Small changes only: one task = one pull request under about 400 changed lines, excluding generated files and tests.

5. No code is done without tests that prove it, and all existing tests must still pass.

6. Never invent tax rates, ATO rules, award rates or API fields. If the source isn't in the task brief, stop and ask.

7. Never copy code from GPL, AGPL, LGPL, Elastic or Business Source licensed projects.

8. Never touch production data, credentials or infrastructure directly. Agents work in their own branch and sandbox.

9. Never add a new dependency without stating why, its licence, its weekly downloads and its last release date in the pull request.

10. If unsure, write down the assumption in the pull request and flag it for human review rather than guessing silently.

11. Report honestly: if a test was skipped, a step failed, or something is incomplete, say so in the pull request summary.

## 2. Agent team and workflow

Agents work in separate roles so no agent checks its own work, and a human approves every merge.

```

Task brief (human product lead)

   -> Planner agent (splits into small task briefs)

   -> Builder agent (code + tests in its own branch)

   -> CI gates (tests, scans, budgets)

        - fails  -> back to Builder agent

        - all green -> Reviewer agents (security + quality)

   -> Human approval (tech lead; accountant adviser for ledger/tax/payroll)

        - changes requested -> back to Builder agent

   -> Merge and deploy (staging, then live behind a feature flag)

```

| Agent | Does | Never does |
| --- | --- | --- |
| Planner | Turns a feature into small task briefs using the template in this handbook; lists dependencies and open questions | Writes production code |
| Builder | Implements one brief in its own branch: code, tests, migration, docs | Merges, changes other modules' internals, skips tests |
| Test agent | Adds missing edge-case, property and isolation tests; tries to break the feature | Edits the code under test to make tests pass |
| Security reviewer | Reviews every pull request against the Security section; runs scans | Approves its own fixes |
| Code reviewer | Reviews against the checklist: architecture, ledger rules, performance, readability | Rewrites the feature itself |
| Docs agent | Updates MODULE.md, API docs and help articles | Changes code |

**Human checkpoints:** the product lead approves every brief before building; the tech lead approves every merge; the accountant adviser approves anything touching the ledger, tax or payroll; production deploys and infrastructure changes are always human-triggered.

**Context every agent gets:** this handbook, the module's MODULE.md, the relevant OpenAPI section, the task brief, and the database schema for the tables involved. Agents start each task from a clean context rather than one long chat.

## 3. Architecture standards

We build a modular monolith first: one deployable backend split into strict domain modules, so we move fast now and can split out services later without rewrites.

**Structure**

- One backend (NestJS, TypeScript strict mode) with one folder per domain: `ledger`, `contacts`, `sales`, `purchases`, `banking`, `tax`, `payroll`, `inventory`, `assets`, `projects`, `reporting`, `ai`, `identity`, `billing`.

- Each module has layers: `api` (controllers, DTOs) → `application` (use cases) → `domain` (entities, rules, no framework code) → `infrastructure` (database, external APIs).

- Modules talk to each other only through their public interface or domain events. Never import another module's internals or query its tables.

- Only the `ledger` module writes journal entries. Other modules call `ledger.post(...)` with a balanced entry.

- Business rules live in the `domain` layer with no database or HTTP calls, so they can be unit tested in milliseconds.

**Multi-tenancy**

- Shared database, shared schema; every tenant-owned table has `organisation_id NOT NULL`.

- Tenant context comes from the authenticated session only, never from a request body or query string.

- PostgreSQL row-level security is on for every tenant table as a second safety net behind application filters.

- Large customers can later move to a dedicated database without code changes (tenant routing in one place).

**Consistency and events**

- A business action and its ledger posting happen in one database transaction.

- Side effects (emails, webhooks, ATO lodgement, bank sync) go through a transactional outbox table, then a worker publishes them. Never call an external API inside a database transaction.

- Every event and job is idempotent: running it twice gives the same result.

- Every write endpoint accepts an `Idempotency-Key` header so retries never double-post an invoice or payment.

**Decisions**

- Any architecture change (new module, new datastore, new external provider) needs an Architecture Decision Record in `docs/adr/` approved by the tech lead before code is written.

- Prefer boring, proven technology. No new language, database or framework without an ADR.

## 4. Ledger and accounting rules

The ledger is the product: if its numbers are ever wrong, nothing else matters. These rules are enforced in code, in the database and in tests.

**Double-entry**

- Every journal entry has at least two lines and the sum of debits equals the sum of credits, per currency. A database constraint (deferred trigger) rejects any unbalanced entry.

- Posted entries are immutable. Corrections are reversing entries linked to the original. Drafts can change; posted cannot.

- Every line carries: organisation, account, amount, currency, tax code, source document type and ID, posting date, created by, created at.

- Account balances are derived from journal lines. Cached balances may exist for speed but are rebuilt and checked against lines nightly.

**Money**

- Store money as integer minor units (cents) in `BIGINT` with a currency code, or `NUMERIC(19,4)` where four decimals are needed (unit prices, FX). Never floating point anywhere: database, API, frontend.

- Use one shared `Money` type in the backend and frontend; raw numbers for money are rejected in code review.

- Rounding: round at line level using round-half-even unless the accountant adviser specifies otherwise for a tax rule; record any rounding difference to a rounding account so the entry still balances.

**GST**

- Tax codes are data (a table with effective dates), not hard-coded numbers. GST 10% is a row, not a constant.

- Support tax-inclusive and tax-exclusive pricing, GST-free, input-taxed and out-of-scope codes, and both cash and accrual BAS reporting.

- Every BAS figure must trace back to the exact transactions that make it up (drill-down).

**Periods and dates**

- Financial years follow the organisation's settings (Australian default 1 July to 30 June).

- Locked periods reject new postings unless the user has the period-unlock permission, and every unlock is audited.

- Store timestamps in UTC; store accounting dates as plain dates in the organisation's time zone.

**Multi-currency**

- Each line stores the transaction currency amount, the base currency amount and the rate used.

- Realised and unrealised FX gains and losses post to dedicated accounts; revaluation is a reversible journal.

**Proof**

- After every test suite run, a check asserts: trial balance nets to zero, balance sheet balances, and BAS totals match their transactions.

## 5. Database standards

PostgreSQL is the single source of truth; every schema change is a reviewed, reversible migration.

**Schema conventions**

- Table names plural snake_case (`invoices`, `journal_lines`); columns snake_case.

- Primary keys are UUIDv7 (time-ordered, safe to expose, good index locality).

- Every table has `created_at`, `updated_at`, `created_by`, `updated_by`; tenant tables have `organisation_id`.

- Use foreign keys, `NOT NULL`, `CHECK` and unique constraints. The database enforces rules, not only the code.

- Use enums or lookup tables for statuses; never free-text statuses.

- Documents (invoices, bills) are not hard deleted once issued: they are voided. Drafts may be deleted.

**Migrations**

- One migration tool (e.g. Prisma Migrate, Drizzle or Knex), migrations checked into the repo, never edited after merge.

- Every migration must be backward compatible with the previous app version (expand, migrate data, then contract in a later release) so deployments have zero downtime.

- No long table locks on large tables: create indexes `CONCURRENTLY`, backfill in batches.

- Migrations run in CI against a copy of realistic data volume before merge.

**Row-level security and access**

- RLS policy on every tenant table: `organisation_id = current_setting('app.org_id')::uuid`.

- The app connects with a role that cannot bypass RLS. Admin and migration roles are separate and never used by the app.

- Each request sets the tenant inside its transaction; connection pooling must not leak tenant settings between requests.

**Indexing and queries**

- Every foreign key and every column used in a `WHERE` or `ORDER BY` on large tables has an index starting with `organisation_id`.

- No `SELECT *` in application code; no N+1 queries (detected by tests that count queries per request).

- Any query over 100 ms in staging is logged and must be fixed or justified.

- Large reporting queries run on a read replica, never the primary.

**Audit**

- An append-only `audit_log` records who did what, when, from which IP and device, with before and after values for financial records.

- Audit logs are retained at least as long as financial records and cannot be edited by any app role.

## 6. API standards

The API is written contract-first: the OpenAPI spec is agreed before code, and our own web and mobile apps use the same public API as partners.

- **Contract first:** update `openapi.yaml`, get review, then generate types and clients from it. CI fails if code and spec disagree.

- **Style:** REST, JSON, plural nouns (`/v1/invoices`), actions as sub-resources (`POST /v1/invoices/{id}/void`). Field names camelCase in JSON.

- **Versioning:** URL major version (`/v1`). Breaking changes only in a new version; old versions supported at least 12 months with a published deprecation date.

- **Errors:** RFC 9457 Problem Details (`type`, `title`, `status`, `detail`, `errors[]` per field). Never leak stack traces, SQL or internal IDs of other tenants.

- **Validation:** every request body validated against a schema at the edge (class-validator or Zod). Reject unknown fields.

- **Money in the API:** amounts as strings in decimal form with a currency code (`{"amount":"1250.00","currency":"AUD"}`), never JSON floats.

- **Pagination:** cursor-based (`?limit=50&cursor=...`), maximum 200 per page. Never unbounded lists.

- **Filtering and sorting:** whitelisted fields only.

- **Idempotency:** `Idempotency-Key` header required on all POST endpoints that create money movements; keys stored 24 hours with the original response.

- **Concurrency:** `ETag` / `If-Match` on updates so two users can't overwrite each other's changes.

- **Rate limits:** per user, per organisation and per API client, with `429` and `Retry-After` headers.

- **Webhooks:** signed with HMAC-SHA256 and a timestamp, retried with exponential backoff for 72 hours, with a delivery log customers can see.

- **Partner access:** OAuth 2.0 authorisation code with PKCE, granular scopes (`invoices.read`, `invoices.write`), tokens revocable by the customer.

- **Bulk:** large imports (opening balances, Xero migration) run as async jobs with a job status endpoint, never one huge request.

## 7. Security standards

We build to OWASP ASVS Level 2 everywhere and Level 3 for identity, the ledger, payments, payroll and ATO lodgement. Security findings block a merge; they are never "fixed later".

**Identity and login**

- Use a proven identity provider (e.g. Auth0, Clerk, AWS Cognito or Keycloak); never write our own password or token crypto.

- MFA mandatory for every user (authenticator app or passkey); SMS only as a last-resort fallback.

- Passkeys (WebAuthn) supported from launch.

- Passwords: minimum 12 characters, checked against breached-password lists, no forced periodic changes.

- Sessions: short-lived access tokens (15 minutes), rotating refresh tokens, HttpOnly + Secure + SameSite cookies on web, idle timeout 30 minutes, users can see and end active sessions.

- Lock or slow down after repeated failed logins; alert users by email on new device sign-in.

**Authorisation**

- Role-based access (Owner, Admin, Accountant, Bookkeeper, Payroll admin, Read-only, Invoice-only) with permission checks in the application layer on every use case, not just hidden buttons.

- Deny by default. Every endpoint declares its required permission; a CI test fails any endpoint without one.

- Sensitive actions (bank details change, payroll run, BAS lodgement, period unlock, adding a user) need re-authentication and are audited; changing supplier bank details notifies all admins (stops invoice fraud).

**Tenant isolation**

- Application filter + row-level security + automated tests that try to read another tenant's data on every endpoint.

- Object IDs are UUIDs, but access is always checked; never trust that an ID is unguessable.

- Files in object storage are stored under the tenant path and served only via short-lived signed URLs.

**Encryption and data protection**

- TLS 1.2 minimum (1.3 preferred) everywhere, HSTS enabled.

- Encryption at rest for database, backups and files using a key management service (AWS KMS).

- Field-level encryption for tax file numbers, bank account numbers and identity documents; mask them in the UI and logs (show last 3 digits).

- All customer data hosted in Australia (AWS Sydney) including backups.

**Secrets**

- Secrets only in a secrets manager (AWS Secrets Manager); never in code, `.env` files in the repo, logs, tickets or AI prompts.

- Secret scanning on every commit (e.g. gitleaks); a found secret is rotated immediately, not just deleted.

- Least-privilege cloud roles; no long-lived access keys for people or agents.

**Secure coding**

- Parameterised queries only (ORM or query builder); no string-built SQL.

- Output encoding in the frontend; strict Content Security Policy; no `dangerouslySetInnerHTML`.

- CSRF protection on cookie-authenticated endpoints.

- File uploads: type and size checked, virus scanned, never executed, stored outside the web root.

- Server-side request forgery protection for any feature that fetches a URL.

**Supply chain**

- Lockfiles committed; dependency and container scanning (e.g. Dependabot, Snyk or Trivy) on every pull request; critical and high vulnerabilities block merge.

- SAST (e.g. Semgrep, CodeQL) on every pull request; DAST against staging weekly.

- Software bill of materials generated per release; signed container images.

**Logging and monitoring**

- Security events logged centrally: logins, failures, permission denials, exports, admin actions, bank detail changes.

- No personal or financial data in application logs beyond IDs.

- Alerts for unusual activity: mass exports, many failed logins, logins from new countries.

**AI-specific threats**

- Treat text from receipts, bank descriptions, emails and uploaded files as untrusted input to AI models (prompt injection risk). AI output never executes actions directly; it only creates suggestions a user approves.

- AI features get read access scoped to the current tenant only and no access to credentials.

**Assurance**

- Independent penetration test before launch and yearly, plus after major changes.

- Follow the ACSC Essential Eight for our own staff and infrastructure.

- Incident response plan with on-call, a 72-hour internal target for breach assessment, and Notifiable Data Breaches process ready.

- Plan for ISO 27001 or SOC 2 from Phase 4.

## 8. Performance and scalability

Speed is a feature we beat Xero on, so every change is measured against fixed budgets; a pull request that breaks a budget does not merge.

| Measure | Budget | How it's checked |
| --- | --- | --- |
| API read endpoints | p95 under 200 ms | Load test in CI, production monitoring |
| API write endpoints (invoice, payment) | p95 under 400 ms | Load test in CI, production monitoring |
| Standard reports (P&L, balance sheet) for 1 year, 50,000 transactions | under 2 s | Benchmark test |
| BAS report for a quarter | under 3 s | Benchmark test |
| Bank reconciliation screen with 500 unmatched lines | interactive under 1 s | Playwright timing test |
| Web page Largest Contentful Paint | under 2.5 s on 4G mobile | Lighthouse CI |
| Interaction to Next Paint | under 200 ms | Lighthouse CI, real-user monitoring |
| Initial JavaScript bundle | under 250 KB compressed | Bundle size check in CI |
| Database queries per API request | 10 or fewer | Query-count test |

**How agents keep it fast**

- Paginate everything; never load all invoices or transactions into memory.

- Precompute account balances per period (rollup tables refreshed on posting) so reports read summaries, not millions of lines.

- Heavy work (imports, report exports, bank sync, AI processing, emails, PDFs) runs in background jobs with progress shown to the user.

- Cache read-heavy reference data (tax codes, chart of accounts, settings) in Redis with clear invalidation on change; never cache data across tenants under a shared key.

- Use database connection pooling (PgBouncer or RDS Proxy).

- Frontend: code-split by route, lazy load heavy screens, virtualise long tables, optimistic UI for common actions, serve assets from a CDN.

- Generate invoice PDFs once and store them; regenerate only when the invoice changes.

**Scale targets to design for**

- 100,000 organisations, 10 million journal lines per large organisation, 1,000 requests per second at peak (end of month, BAS due dates).

- Load test at 2x those peaks before each major release.

- Stateless app servers so we can add more behind the load balancer; autoscaling on CPU and queue depth.

## 9. Reliability and operations

Target 99.9% monthly availability (about 43 minutes of downtime a month) at launch, 99.95% by Phase 4, with no data loss ever.

**Backups and recovery**

- Point-in-time recovery on the database (recovery point objective: 5 minutes or less).

- Recovery time objective: 1 hour for a full region-level restore.

- Daily encrypted backups copied to a second Australian region, kept for 35 days; monthly snapshots kept 7 years.

- A real restore is tested every month and the result recorded. An untested backup does not count.

**Observability**

- OpenTelemetry tracing, metrics and structured JSON logs on every service, with a request ID that follows a request through API, jobs and external calls.

- Dashboards for the four golden signals: latency, traffic, errors, saturation, plus queue depth and job failures.

- Alerts go to on-call with a runbook link; every alert must be actionable or it is removed.

- Error tracking (e.g. Sentry) on backend, web and mobile with personal data scrubbed.

**Deployments**

- Infrastructure as code (Terraform or AWS CDK) for everything; no manual console changes.

- Every merge to main deploys to staging automatically; production deploys at least daily via blue-green or canary with automatic rollback on error-rate increase.

- Feature flags for unfinished or risky features, so code ships dark and turns on per tenant.

- No deploys in the last 2 days before BAS due dates without approval.

**External dependencies**

- Every external call (bank feeds, ATO, payments, AI models) has a timeout, retry with backoff and jitter, and a circuit breaker.

- If a provider is down, the product keeps working and queues the work; users see a clear status message, never a crash.

**Incidents**

- Severity levels, on-call rota, public status page, and a blameless post-incident review within 5 working days of any Sev 1 or Sev 2.

## 10. Frontend and UX standards

Bookkeepers use this software for hours a day, so the UI must be fast, keyboard-friendly, consistent and impossible to misread.

**Design system**

- One shared component library (React + TypeScript, e.g. built on Radix or shadcn/ui primitives) with design tokens for colour, spacing and type. Agents use existing components; they never hand-style one-off buttons, tables or forms.

- Every component documented in Storybook with its states (loading, empty, error, disabled).

- Mobile app (Flutter) uses the same tokens and naming.

**Accessibility**

- WCAG 2.2 AA minimum: keyboard access to everything, visible focus, labels on every field, colour contrast 4.5:1, no meaning conveyed by colour alone (e.g. negative amounts also show a minus sign).

- Automated accessibility checks (axe) in CI on every page.

**Accounting UI rules**

- Amounts right-aligned in tabular figures, always with currency, negatives clearly marked; never show rounded figures where totals must reconcile.

- Keyboard shortcuts for heavy screens (reconciliation, invoice entry, journals).

- Every destructive or financial action shows a clear confirmation with the amount and what will happen; offer undo where the ledger rules allow.

- Every number in a report can be clicked through to the transactions behind it.

- Empty states explain what to do next; errors explain how to fix them in plain English.

**Forms and data**

- Validate on the client for speed and on the server for truth, with the same rules shared from the schema.

- Autosave drafts; warn before leaving with unsaved changes.

- Server state via one data library (e.g. TanStack Query) with consistent loading, error and retry handling.

**Localisation**

- All text through an i18n library from day one (English first; Tamil, Hindi and Sinhala later), no hard-coded strings.

- Dates, numbers and currency formatted by locale (en-AU default: DD/MM/YYYY, AUD).

- Invoice templates support Unicode scripts and the right fonts for Tamil and other languages.

## 11. Testing and quality gates

AI agents write code fast, so tests are what stop them shipping confident mistakes; no pull request merges unless every gate below is green.

| Test type | What it proves | Tool (example) | Required |
| --- | --- | --- | --- |
| Unit tests | Domain rules (GST, rounding, posting) are correct | Vitest / Jest | 90% line coverage on `domain` layers, 80% overall |
| Property-based tests | Ledger stays balanced for thousands of random transactions | fast-check | Ledger, tax, FX, payroll modules |
| Golden-file tests | BAS, payslips and reports match accountant-verified outputs exactly | Snapshot fixtures signed off by the adviser | Every report and tax form |
| Integration tests | Module + real PostgreSQL + RLS work together | Testcontainers | Every use case |
| Tenant isolation tests | No endpoint returns another organisation's data | Custom suite | Every endpoint, automatically |
| Contract tests | API matches the OpenAPI spec; external providers' mocks match their real APIs | Schemathesis / Pact | Every endpoint and integration |
| End-to-end tests | Main user journeys work in a real browser | Playwright | Top 20 journeys |
| Performance tests | Budgets in the Performance section hold | k6 | Nightly and before release |
| Security scans | No known vulnerabilities, secrets or unsafe code | Semgrep, CodeQL, gitleaks, Trivy | Every pull request |
| Mutation testing | Tests actually catch bugs in critical code | Stryker | Ledger and tax modules, weekly |

**CI pipeline gates (in order)**

1. Type check (TypeScript strict), lint, format.

2. Secret scan and dependency scan.

3. Unit and property tests.

4. Integration, isolation and contract tests.

5. SAST, bundle size, Lighthouse and accessibility checks.

6. Migration check against realistic data.

7. Review by a reviewer agent and a human (see the workflow section).

**Test data rules**

- Never use real customer data in tests or agent prompts. Use generated data from shared factories.

- Keep a realistic demo company (two years of transactions, payroll, multi-currency) for manual testing and demos.

- A failing test is never deleted or skipped to make a build pass without a human's written approval in the pull request.

## 12. AI features inside the product

AI suggests; people decide. No AI feature posts to the ledger, pays money or lodges with the ATO without a user's explicit approval.

- **Suggestions, not actions:** categorisation, reconciliation matches, receipt data and anomaly flags appear as suggestions with an Accept / Edit / Reject choice. Bulk accept is allowed only above a confidence threshold the user sets.

- **Confidence and reasons:** every suggestion shows a confidence level and a one-line reason ("Matched: same amount, payee name similar, 2 days apart").

- **Rules before models:** use deterministic rules and the organisation's own history first; call an LLM only when rules don't decide. This keeps it fast, cheap and explainable.

- **Evaluation sets:** each AI feature has a labelled test set (e.g. 1,000 categorised bank lines, 500 receipts). Accuracy is measured on every model or prompt change; a drop of more than 2 percentage points blocks release.

- **Prompts in code:** prompts are versioned files in the repo, reviewed like code, with tests.

- **Data use:** customer data is never used to train third-party models; use AI providers with zero-retention or no-training terms and Australian or approved data handling. Tell customers clearly in the privacy policy.

- **Minimum data:** send the model only the fields it needs (e.g. description and amount), never tax file numbers, bank account numbers or full customer records.

- **Cost and speed limits:** per-tenant usage limits, caching of repeated requests, timeouts with a graceful "no suggestion" fallback.

- **Learning loop:** user corrections are stored per organisation and improve that organisation's future suggestions.

- **Audit:** every AI suggestion and the user's decision is logged with the model and prompt version.

## 13. Australian compliance rules

Agents implement compliance rules only from a source document linked in the task brief and checked by the accountant adviser; tax and payroll rules change often, so every rate and threshold is stored as dated data, never as a constant.

| Area | What agents must build in | Owner of the source |
| --- | --- | --- |
| Tax invoices | Words "Tax invoice", seller name and ABN, date, item descriptions, GST amount; buyer identity or ABN for sales of $1,000 or more | ATO rules, accountant adviser |
| GST and BAS | Cash and accrual BAS, GST labels traced to transactions, lodgement via SBR once approved | ATO, tech lead |
| Record keeping | Financial records kept and retrievable for at least 5 years; payroll records at least 7 years; nothing auto-deleted earlier | ATO, Fair Work |
| Payroll | STP Phase 2 reporting, super guarantee rate as dated data, payday super rules, leave accruals, award interpretation reviewed by the payroll specialist | ATO, Fair Work, payroll specialist |
| ATO software requirements | Meet the DSP Operational Framework controls (e.g. MFA, encryption, audit logging, data location, security monitoring) | Product lead |
| Privacy | Australian Privacy Principles, privacy policy, data access and correction requests, data export on request | Product lead |
| Data breaches | Notifiable Data Breaches process: assess suspected breaches quickly (law allows at most 30 days) and notify OAIC and affected people when required | Product lead, security |
| Marketing emails | Consent and one-click unsubscribe (Spam Act) | Product lead |
| Subscriptions | Clear pricing, cancellation and refunds under Australian Consumer Law | Product lead |
| E-invoicing | Peppol PINT A-NZ format via an accredited access point | Tech lead |

Before any of these is marked done, the owner confirms it against the current official source and records the source link and check date in the module's `MODULE.md`.

## 14. Repository and code conventions

One monorepo with one way of doing each thing, so every agent produces code that looks like the rest.

```

/apps

  /api            NestJS backend (modular monolith)

  /worker         background jobs (same codebase, separate process)

  /web            React web app

  /mobile         Flutter app

/packages

  /money          shared Money type and rounding

  /contracts      OpenAPI spec + generated types

  /ui             design system components

  /config         lint, tsconfig, test config

/infra            Terraform or CDK

/docs

  /adr            architecture decision records

  /modules        MODULE.md per module

AGENTS.md         this handbook

BUILD_PLAN.md     the build plan

```

- **Package manager and build:** pnpm workspaces with Turborepo (or Nx) for cached builds.

- **TypeScript:** `strict: true`, no `any`, no `@ts-ignore` without a comment and reviewer approval.

- **Style:** ESLint + Prettier, enforced in CI; no style debates in review.

- **Naming:** files kebab-case, classes PascalCase, functions camelCase, database snake_case.

- **Errors:** typed domain errors mapped to Problem Details at the API edge; never swallow errors silently.

- **Comments:** explain why, not what; every non-obvious accounting rule cites its source.

- **Branches:** short-lived feature branches from `main`; branch name `agent/<ticket-id>-short-name`.

- **Commits:** Conventional Commits (`feat(sales): add credit notes`), which drive the changelog.

- **Pull requests:** template with: what changed, why, how it was tested, risks, assumptions, screenshots for UI, and a checklist of this handbook's sections that apply.

- **Docs:** update `MODULE.md`, the OpenAPI spec and user-facing help in the same pull request as the code.

## 15. Agent task brief and review checklist

Most AI mistakes come from vague briefs, so every task given to a builder agent uses this template, and every pull request is reviewed against the checklist below.

**Task brief template (copy for each task)**

```

TASK ID: SALES-042

GOAL: Users can issue a credit note against an existing invoice.

MODULE: sales (read docs/modules/sales/MODULE.md first)

IN SCOPE:

  - POST /v1/invoices/{id}/credit-notes endpoint

  - Ledger posting: reverse revenue and GST for the credited lines

  - Credit note PDF using the existing invoice template

OUT OF SCOPE:

  - Refund payments (separate task SALES-043)

RULES AND SOURCES:

  - Ledger rules: AGENTS.md section "Ledger and accounting rules"

  - GST treatment: <link to ATO page + adviser note>

DATA MODEL: <tables and fields involved>

ACCEPTANCE CRITERIA:

  1. Credit note total cannot exceed invoice balance

  2. Journal entry balances and reverses GST correctly

  3. Invoice shows remaining balance after credit

TESTS REQUIRED: unit, property (ledger balance), integration,

  tenant isolation, one Playwright journey

PERFORMANCE: endpoint p95 under 400 ms

SECURITY: permission sales.credit_notes.create; audited

DONE WHEN: all CI gates green + reviewer agent + human approval

```

**Review checklist (reviewer agent and human)**

- [ ] Does it do exactly what the brief asked, and nothing extra?

- [ ] Ledger: every posting balanced, no posted entry edited, Money type used, rounding handled

- [ ] Tenancy: tenant comes from the session, RLS active, isolation test added

- [ ] Security: permission check present, input validated, no secrets, no sensitive data in logs, audit entry for sensitive actions

- [ ] API: matches OpenAPI, Problem Details errors, pagination, idempotency key on money endpoints

- [ ] Database: migration backward compatible, indexes for new queries, no N+1

- [ ] Performance: within budget; heavy work moved to a background job

- [ ] Tests: required types present, meaningful assertions, nothing skipped

- [ ] UI: design system components, accessible, loading/empty/error states, amounts formatted correctly

- [ ] Docs: MODULE.md, OpenAPI and changelog updated

- [ ] Dependencies: any new one justified and licence-checked

- [ ] Assumptions listed and nothing invented (tax rates, API fields, rules)

## 16. Definition of done and release checklist

A task is done only when a customer could safely use it in production; a release goes out only when every item below is ticked.

**Definition of done (every task)**

- [ ] Acceptance criteria in the brief all met

- [ ] All CI gates green, no skipped tests

- [ ] Reviewer agent and human reviewer approved

- [ ] Accountant adviser signed off any ledger, tax or payroll change

- [ ] Behind a feature flag if not ready for all customers

- [ ] Docs, OpenAPI and changelog updated

- [ ] Monitoring and alerts exist for any new job or integration

**Release checklist (every production release with new features)**

- [ ] Load test passed at 2x peak; performance budgets hold

- [ ] Security scans clean; no open critical or high findings

- [ ] Migrations tested on a production-size copy; rollback plan written

- [ ] Backup taken and last restore test passed this month

- [ ] Golden-file tests (BAS, payslips, reports) pass unchanged or changes signed off

- [ ] Not within 2 days of a BAS or STP due date (or approved)

- [ ] Release notes and help articles ready; support team briefed

- [ ] On-call engineer confirmed for 24 hours after release

**Before public launch (one time)**

- [ ] Independent penetration test done and findings fixed

- [ ] ATO DSP requirements met for the features being launched

- [ ] Privacy policy, terms, and data breach process live

- [ ] Pilot customers ran their books for at least one full month and one BAS period without a ledger error

- [ ] Disaster recovery drill completed within the 1-hour target
