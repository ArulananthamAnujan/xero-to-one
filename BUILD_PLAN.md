# Accounting Platform Build Plan

Version: 1.1 · 9 Oct 2026 · Owner: Anujan (product lead and tech lead)

## Change log

### Version 1.1 — 9 Oct 2026 (DOCS-000)

Only approval Parts A–G and their accepted report findings are incorporated; original section numbers remain. The study report is historical analysis, not an implementation authority where this version or the approval differs.

- **B1 / R11, R12, R33:** explicitly include Foundation ledger/reversal/locks/trial balance, opening imports, organisation/membership, contacts/ABN, invoice/credit/void/PDF/email, bills/approval, manual receipts/payments/allocations, GST kernel/dated codes, attachments and web screens; keep BAS reporting/lodgement in Phase 2.
- **B1 / R12, R29:** move quotes SALES-007/008, recurring SALES-009/010, claims PURCHASES-005/006, OCR/AI AI-001–003 and partner/webhook PARTNER-001–003 into Phase 2; replace the “day one” partner-API promise. Publish a superseding Phase 1 task-name/dependency index; no expanded briefs authorised by that index.
- **B2 / R10, C4:** AUD-only Phase 1/pilot with retained currency/FX columns and `NUMERIC(24,10)` rates; multi-currency stays Phase 3 and cannot be promised on pricing pages before shipping.
- **B3 / R16, Part A / R34:** adviser reviews at the end; provisional decisions and readable pack are prepared before/during implementation; Anujan approves rules before code, flags stay in place, and adviser register sign-off is a hard gate before real parallel pilot bookkeeping. Pilot businesses will be named separately.
- **R13:** Peppol consistently Phase 3, matching the roadmap; own-access-point versus partner choice remains open. **R14:** payroll specialist engaged part-time from Phase 3 for design/registration, implementation Phase 4.
- **B4 / R17:** recovery RPO <=5 minutes/RTO 1 hour are engineering targets only. **B5 / R18:** Australian AWS data/identity/log/backup residency; overseas subprocessors need Anujan's written ADR approval.
- **B6 / R15:** report-only/manual BAS launch allowed if DSP approval is delayed; direct lodgement disabled until approved; DSP security controls mandatory from Phase 1.
- **E / R30:** queue/cache technology remains pending ADR-0004 (Valkey proposed); diagram and stack description no longer imply Redis is settled.
- **A, C1–C3, E–G / R05, R30:** Anujan is product lead and tech lead and chooses identity after the comparison ADR; other human roles are future hires; tenant-local users with later multi-client isolation design; next-three brief limit and per-brief approval; Part D must merge before Part E work; current external/infrastructure/spend prohibitions and session status retained.

- **Revised approval Part A (9 Oct 2026):** Anujan holds both lead roles; every PR receives separate security, code and test agent reviews before his approval; no current accountant appointment or per-change adviser dependency. Distinguish approved development progression from accountant-gated real use; mark human hires as future and reassess the staffing-dependent schedule.
- **Revised approval Part A2 (new decision, not a study finding):** use documented Xero accounting behaviour/workflows as the reference; record linked behaviour and deviations in each accounting decision and future import field mappings; retain ATO authority and prohibit copied presentation, unapproved branding and live-account/API study.

## 1. Goal and how we beat Xero

We build an Australian-first cloud accounting platform that matches Xero's core and wins on price, AI automation and local compliance. We do not merge the open-source projects; we study them and write our own code.

Xero's publicly documented accounting behaviour and workflows are the reference, under AGENTS.md section 2. Each accounting decision records linked Xero behaviour or an explicit documentation gap and explains deviations; ATO sources take precedence. Maintain future import field mappings in `docs/accounting/xero-mapping.md`. Use original design, branding and text; no Xero code/presentation copying, unapproved product/marketing use of its name/logo or "like Xero" claims, login, trial, scraping or API study. Research topics do not expand the approved phase scope.

Where we beat Xero:

- **Price:** no invoice or bill caps on the entry plan. Phase 1 and pilot are AUD only; multi-currency is planned for Phase 3 and pricing pages must not promise it until it ships.

- **AI bookkeeping:** auto-categorise bank transactions, read receipts and bills, and suggest reconciliations with a confidence score.

- **Australia built in, not added on:** BAS, GST, STP Phase 2 payroll and super handled natively, not through paid add-ons.

- **Community focus:** Tamil, Hindi and Sinhala invoice templates and language support for the South Asian small-business market we already serve.

- **Accountant tools:** one dashboard for practices to manage many clients, with bulk BAS lodgement.

- **Open partner API from Phase 2:** so other apps (POS, Shopify, Square) connect easily. The internal web app still uses the contract-first API in Phase 1; partner access and webhooks are deferred.

## 2. How we use the open-source projects

Each project is a reference design for specific modules: we read its data model and workflows, then write our own implementation. No code is copied unless its licence is MIT or Apache and the tech lead approves it.

| Project | GitHub | Study it for | Stack |
| --- | --- | --- | --- |
| Bigcapital | bigcapitalhq/bigcapital | Ledger, invoices, bills, banking, inventory, financial reports | Node, TypeScript, React |
| Frappe Books | frappe/books | Clean double-entry engine, chart of accounts, journal design | Vue, Electron |
| Akaunting | akaunting/akaunting | Modular app/add-on system, multi-company | Laravel, PHP |
| ERPNext | frappe/erpnext | Payroll, fixed assets, projects, multi-currency, period close | Python, Frappe |
| Odoo | odoo/odoo | Bank reconciliation UX, accountant workflows | Python |
| Invoice Ninja | invoiceninja/invoiceninja | Invoice templates, recurring invoices, client portal, payments | Laravel, PHP |
| Crater | crater-invoice/crater | Estimates, expenses, simple invoicing UI | Laravel, Vue |
| GnuCash / Beancount / LedgerSMB | Gnucash/gnucash, beancount/beancount, ledgersmb/LedgerSMB | Accounting correctness rules, trial balance, reconciliation logic | C, Python, Perl |

Licence rules for the team:

1. Before opening any repo, record its licence in our tracker.

2. GPL, AGPL, LGPL, Elastic or Business Source licences: read for ideas only. Never paste code, file structure or SQL from them.

3. MIT or Apache: code may be reused with attribution, after tech lead sign-off.

4. Each developer writes a short design note in their own words before coding a module (clean-room record).

## 3. Tech stack and architecture

One multi-tenant platform where every module writes through a single ledger engine, so the books always balance.

```

Clients:      Web app (React, TS) | Mobile app (Flutter) | Public API (REST + webhooks)

                               |

Gateway:      API gateway: login, MFA, roles, rate limits

                               |

Modules:      Invoicing | Bills | Banking | GST & BAS | Payroll | Reporting | Inventory | AI service

                               |                         <-> External services:

Core:         CORE LEDGER ENGINE (balanced journals,          ATO (BAS, STP via SBR)

              audit log, period locks)                         Bank feeds (CDR provider)

                               |                               Payments (Stripe, PayTo)

Data:         PostgreSQL (AWS Sydney) | File storage           Peppol (e-invoices)

              (receipts, PDFs) | Job queue (ADR-0004 pending)             AI model API (OCR, LLM)

```

- **Backend:** Node.js with TypeScript (NestJS). Same language as Bigcapital, so the team can study it easily.

- **Web:** React with TypeScript. **Mobile:** Flutter, which the team already uses.

- **Database:** PostgreSQL, every row tagged with its organisation, row-level security on.

- **Background jobs:** Redis-compatible queue for bank feed syncs, ATO lodgements and emails; Valkey proposed, ADR-0004 pending. No queue technology is installed by this docs change.

- **Hosting:** AWS Sydney primary, separate test and production; all customer data, backups, logs and identity data remain in Australian AWS regions. An overseas subprocessor, including AI/email, needs Anujan's written ADR approval before use. RPO <=5 minutes and RTO 1 hour are engineering recovery targets, not customer promises.

- **Phase 1 currency:** AUD only, including pilot. Keep currency/FX columns, use `NUMERIC(24,10)` for `fx_rate`, and reject any non-AUD transaction or base currency at the posting gateway. Multi-currency functionality stays in Phase 3.

- **Pending design decisions:** ADR-0002 retains tenant-local users and describes isolated multi-client access for the Phase 4 accountant hub. ADR-0003 compares AWS Cognito (`ap-southeast-2`) and self-hosted Keycloak on MFA/passkeys, step-up, AU residency, operations, 1,000/100,000-user costs and lock-in; Anujan chooses. ADR-0004 proposes Valkey; no datastore/provider is installed or changed by this document.

- **AI:** an LLM API plus OCR for receipts; every AI suggestion needs a user's approval before it posts.

## 4. Module map

Sixteen modules, built on one shared ledger: every module posts journal entries to the ledger and never stores balances of its own.

| Module | What it does | Reference project | Australian requirement | Phase |
| --- | --- | --- | --- | --- |
| Core ledger | Chart of accounts, balanced drafts, posting, reversals, trial balance, period lock, opening balances import | Frappe Books, Beancount | AU standard chart of accounts template | 1 |
| Contacts | Customers, suppliers, ABN lookup | Bigcapital | ABN validation via ABR lookup | 1 |
| Invoicing | Invoices, credit notes including paid invoices, voids, PDF/email delivery; quotes and recurring later | Invoice Ninja, Bigcapital | Tax invoice rules (ABN, GST shown) | 1; quotes/recurring 2 |
| Bills and expenses | Bills, approval, attachment upload; expense claims and AI receipt reading later | Bigcapital, Crater | GST credits on purchases | 1; claims/OCR 2 |
| Banking | Manual receipt/payment records and allocations; bank feeds, statement import, reconciliation later | Odoo, GnuCash | CDR provider for live bank feeds (e.g. Basiq) | 1 manual; 2 feeds/reconciliation |
| Payments | Manual customer receipts/supplier payments with allocation in Phase 1; pay-now links, card and PayTo in Phase 2 | Invoice Ninja | PayTo and BPAY options for provider payments | 1 manual records; 2 payment execution |
| GST and BAS | GST calculation kernel and dated codes; BAS reports and lodgement later | ERPNext (tax engine) | DSP security controls from Phase 1; direct ATO lodgement only once approved | 1 kernel; 2 BAS |
| Reporting | P&L, balance sheet, cash flow, aged receivables/payables | Bigcapital | AU report formats | 2 |
| Inventory | Items, stock levels, cost of goods | Bigcapital | None specific | 3 |
| Fixed assets | Asset register, depreciation | ERPNext | ATO depreciation methods, instant asset write-off | 3 |
| Projects | Time tracking, job costing | ERPNext | None specific | 3 |
| Payroll | Pay runs, leave, payslips, super | ERPNext | STP Phase 2, Fair Work awards, super clearing | 4 |
| Multi-currency | Foreign invoices, revaluation | ERPNext, GnuCash | RBA exchange rates | 3 |
| E-invoicing | Send and receive Peppol invoices | None (build from spec) | Peppol access point or accredited partner, decision pending | 3 |
| AI assistant | Categorise, reconcile, answer questions about the books | None (our differentiator) | Explainable suggestions, human approval | 2 onward |
| Accountant hub | Multi-client dashboard, bulk BAS | Odoo | Tax agent workflows | 4 |

## 5. Phased roadmap

The paid MVP target remains month 8 with invoicing, bills, bank feeds, GST/BAS and AI categorising; payroll comes last because it carries the most compliance risk. These remain estimates subject to the gates below. If DSP approval is delayed at Phase 2 end, launch may use BAS reports for manual lodgement, with direct lodgement disabled; DSP security controls remain requirements from Phase 1.

| Phase | Months | Name | Scope | Gate to pass before next phase |
| --- | --- | --- | --- | --- |
| 1 | 1–4 | Foundation | AUD-only approved scope below, including GST kernel, opening imports and manual allocations | Adviser signs off decisions register before any real pilot bookkeeping; parallel pilots then run 1 month; trial balance tests pass |
| 2 | 5–8 | Launch MVP | Bank feeds/reconciliation, BAS reports/lodgement, reports, provider payments, AI categorising and explicit deferrals below | Adviser review/sign-off covers BAS decisions and examples; DSP approval for direct lodgement, or report-only manual-BAS launch with direct lodgement disabled; DSP security controls still mandatory |
| 3 | 9–13 | Grow | Inventory, fixed assets, projects, multi-currency, mobile app, Peppol e-invoicing | Paying customers stable; payroll design approved |
| 4 | 14–18 | Payroll and practices | STP Phase 2 payroll, super, leave, awards, accountant hub, Xero import tool, ISO 27001 start | — |

Phase progression still requires Anujan's approval and the applicable engineering gates. Accountant review and real-pilot milestones are real-use/release gates, not blockers to separately approved development with synthetic data: do not wait for the later accountant engagement to develop. No real pilot bookkeeping is permitted before register sign-off. The month ranges assumed the planned human team below and must be reassessed with Anujan.

**Approved Phase 1 scope:** ledger (chart of accounts, balanced journal drafts/posting, reversal, period lock and trial balance); opening balances import; organisation and membership setup; contacts with ABN Lookup; invoices; credit notes including against paid invoices; voids; invoice PDF and email delivery; bills and approval; manual customer receipts and supplier payments with allocation; GST calculation kernel with dated tax codes; bill-attachment file upload; web app for all of these. Manual payment records do not execute external payments. Adviser-pack BAS-style examples are review material, not a Phase 1 BAS report feature.

**Phase 2 deferrals (removed from the operative Phase 1 backlog):**

| Task IDs | Deferred work |
| --- | --- |
| SALES-007, SALES-008 | Quote lifecycle/conversion and UI |
| SALES-009, SALES-010 | Recurring drafts/schedules and UI |
| PURCHASES-005, PURCHASES-006 | Expense claims and receipt/claim workflow |
| AI-001, AI-002, AI-003 | Receipt OCR suggestion contract, adapter and UI; all AI work |
| PARTNER-001, PARTNER-002, PARTNER-003 | Partner OAuth/scopes, partner rate-limit/docs work and webhooks; Phase 1 user/org rate limits remain required |

**Pilot model and accounting gate:** Anujan will name 3–5 businesses separately. No real pilot bookkeeping before adviser review/sign-off of the decisions register. After sign-off, pilots run in parallel with their existing bookkeeping system; this product is not their legal system of record during pilot. Until then, use synthetic demo data and feature-flag provisional accounting code.

**Operative Phase 1 task index:** the following names/dependencies supersede study report §7 for Phase 1. They are not approved expanded briefs. Every build task still needs Anujan's approval of exact files/functions, at least five example-based acceptance criteria, named tests and exclusions; expand at most the next three tasks. Reference approvals to adviser fixtures in the old index now mean source-backed provisional decisions approved by Anujan, with end-review/hard pilot gate retained. Independent ledger/security reviewer approval remains required.

| ID | Task name | Task dependencies / approval gates |
| --- | --- | --- |
| REPO-001 | Establish pnpm/Turborepo skeleton and handbook copies | DOCS-000 merged; approved ADR-0001/0002/0003/0004/0006 |
| CI-001 | Add type/lint/format, secret and dependency gates | REPO-001 |
| CI-002 | Add unit/property coverage and real-PostgreSQL integration harness | CI-001 |
| MONEY-001 | Define immutable Money parse/format, currency and range rules | CI-002; Anujan-approved provisional Money/rounding decision |
| MONEY-002 | Implement exact scaled multiplication and explicit rounding-policy interface | MONEY-001; Anujan-approved provisional rounding decision |
| LEDGER-001 | Write core posting contract and module ownership documentation | MONEY-001, ADR-0002, ADR-0005 |
| IDENTITY-001 | Create organisation, tenant-local users and membership migration | LEDGER-001 |
| LEDGER-002 | Create accounts and periods migration | IDENTITY-001 |
| TAX-001 | Create dated tax-code and tax-settings history schema | LEDGER-002 |
| LEDGER-003 | Create journal header/line migration and ledger indexes | TAX-001 |
| SEC-001 | Apply FORCE RLS and transaction-local context to core tables | LEDGER-003 |
| AUDIT-001 | Create append-only audit schema and protected append path | SEC-001 |
| LEDGER-004 | Implement header/line mutation guards and deferred balance constraints | AUDIT-001 |
| LEDGER-005 | Implement atomic ledger post application interface | LEDGER-004 |
| LEDGER-006 | Implement period lock/unlock command | LEDGER-005 |
| LEDGER-007 | Implement full reversal command | LEDGER-006 |
| LEDGER-008 | Expose paginated journal and trial-balance read interfaces | LEDGER-007 |
| API-001 | Integrate provider session validation and active-tenant selection | SEC-001 |
| API-002 | Implement step-up/session lifecycle and provider policy configuration | API-001 |
| API-003 | Implement permission guard and missing-permission CI rule | API-002 |
| API-004 | Implement durable command idempotency and response replay | API-003, LEDGER-005 |
| API-005 | Add Problem Details, If-Match and bounded list infrastructure | API-004 |
| OUTBOX-001 | Create transactional outbox and consumer inbox schemas | API-004, AUDIT-001 |
| OUTBOX-002 | Implement tenant-scoped dispatcher and lease recovery | OUTBOX-001 |
| OUTBOX-003 | Implement worker dedupe/retry/dead-letter status | OUTBOX-002 |
| OPS-001 | Describe sandbox network/database IaC and human apply workflow | CI-002; separate authorisation before any infrastructure apply |
| OPS-002 | Add redacted logs, trace IDs and service health | OPS-001; separate authorisation before any infrastructure apply |
| WEB-001 | Create app shell, locale setup and login/tenant navigation | API-005 |
| WEB-002 | Add shared form/table/Money display components and Storybook states | WEB-001 |
| ORG-001 | Add organisation bootstrap/settings and invitation command contracts | API-003, WEB-002 |
| ORG-002 | Build organisation/settings and membership web views | ORG-001 |
| COA-001 | Add chart template import and account maintenance API | LEDGER-008, API-005 |
| COA-002 | Build chart, manual-journal and trial-balance views | COA-001, WEB-002 |
| OPEN-001 | Add staged opening-balance import command | COA-001, LEDGER-007 |
| CONTACTS-001 | Define contacts contract and migration | ORG-001, API-005 |
| CONTACTS-002 | Implement contact CRUD and paginated search | CONTACTS-001 |
| CONTACTS-003 | Implement ABN Lookup adapter with provenance and outage handling | CONTACTS-002 |
| CONTACTS-004 | Build contact list/editor and lookup confirmation | CONTACTS-003, WEB-002 |
| TAX-002 | Implement invoice/bill tax calculation kernel | TAX-001, MONEY-002 |
| SALES-001 | Define invoice/document-number contracts and schema (quote schema deferred) | CONTACTS-002, TAX-002 |
| SALES-002 | Implement invoice draft commands | SALES-001, API-005 |
| SALES-003 | Implement invoice issue command | SALES-002, API-004, OUTBOX-001 |
| SALES-004 | Build invoice draft/issue/list/detail screens | SALES-003, WEB-002 |
| DOCS-001 | Generate immutable invoice PDF with approved template | SALES-003, OUTBOX-003 |
| DOCS-002 | Add email delivery adapter and delivery status | DOCS-001 |
| SALES-005 | Implement credit-note and void commands | SALES-003, LEDGER-007 |
| SALES-006 | Build credit/void confirmation and document views | SALES-005, SALES-004 |
| PURCHASES-001 | Define bill schema and contracts (expense-claim schema deferred) | CONTACTS-002, TAX-002 |
| PURCHASES-002 | Implement bill draft/version commands | PURCHASES-001, API-005 |
| PURCHASES-003 | Implement bill approval/post command | PURCHASES-002, OUTBOX-001, LEDGER-005 |
| PURCHASES-004 | Build bill list/editor/approval screens | PURCHASES-003, WEB-002 |
| FILES-001 | Implement receipt upload quarantine and clean-file access | PURCHASES-001, OPS-001 |
| BANKING-001 | Define manual receipts/payments and allocation contract/schema | SALES-005, PURCHASES-003 |
| BANKING-002 | Implement manual receipt and supplier payment commands | BANKING-001, API-004 |
| BANKING-003 | Build manual settlement and allocation UI | BANKING-002, SALES-004, PURCHASES-004 |
| LEDGER-009 | Add ledger-owned period rollup update/rebuild and discrepancy check | LEDGER-008, BANKING-002 |
| OPS-003 | Define backups, restore and human-operated release runbooks/IaC | OPS-002, FILES-001; separate authorisation before any infrastructure apply |
| QA-001 | Add independent ledger state-machine/mutation and isolation regressions | LEDGER-009; all Phase 1 financial commands |
| QA-002 | Add top-20 user-journey and performance/accessibility acceptance suite | QA-001; all approved Phase 1 screens (exclude deferred journeys) |
| SEC-002 | Complete ASVS evidence map and release security review | OPS-003; all approved Phase 1 endpoints |
| PILOT-001 | Prepare pilot reconciliation pack and Foundation gate review | QA-002, SEC-002; adviser register sign-off; Anujan pilot nomination |

**Authorised next work and gates:** DOCS-000 version 1.1 changes must be reviewed and merged by Anujan first. Then, in order, each as its own PR: (1) draft ADR-0001 scope, ADR-0002 data/tenancy, ADR-0003 identity, ADR-0004 Valkey queue, ADR-0006 residency/recovery for Anujan's approval; (2) expand/approve/build REPO-001, CI-001 and CI-002; (3) write initial provisional rounding, draft/post/reversal, lock and credit-note decisions for Anujan to approve before code; (4) expand/approve/build MONEY-001 and MONEY-002 with configurable rounding; (5) create compliance source register SRC-01–09, marking unconfirmed sources. The later ledger-specific ADR-0005 named in the study remains a prerequisite proposal for its dependent ledger tasks, not permission to start it in Part D. All later tasks remain gated by approved expanded briefs. No infrastructure apply, account creation, spend, email or application to ATO/banks/providers is authorised yet. Do not add dependencies beyond study §5 without the full golden-rule-9 evidence; named dependencies still require that evidence.

## 6. Team structure

The existing month estimates assumed a core human team of 6 to 8, with additional payroll and e-invoicing specialists. Those hires are not confirmed; do not treat those estimates as a validated schedule for Anujan and agents. Reassess capacity with Anujan before committing dates. Each module needs one accountable owner.

Product lead and tech lead: Anujan, who approves every brief, ADR and merge. Wherever this plan or the historical report says tech lead, read Anujan. Security, code and test reviewer agents review every PR in sessions separate from the builder before it reaches Anujan. Accountant adviser: none for now; engage later for finished-product review. Development does not wait for adviser approval; real pilot bookkeeping still requires accountant sign-off of the decisions register. The remaining roles below are future hires, not current staff.

| Role | Status / planned count | Owns |
| --- | --- | --- |
| Product lead (Anujan) | 1 | Scope, priorities, compliance applications, final sign-off |
| Tech lead / architect (Anujan, same person as product lead) | Current | Architecture, ledger engine, code review, licence approvals |
| Backend developers | Future hire; 2 | Contacts, invoicing, bills, banking, GST/BAS, reporting APIs |
| Frontend developer | Future hire; 1 | Web app UI, design system, invoice templates |
| Mobile developer | Future hire; 1 (from Phase 3) | iOS/Android app: receipts, invoices, approvals |
| AI/ML engineer | Future hire; 1 | Categorisation, receipt reading, reconciliation suggestions |
| QA and accounting tester | Future hire; 1 | Test plans, accounting correctness, regression suite |
| Accountant adviser (CPA/CA, part-time) | Future hire; 1 | End review of built product, decisions register and readable pack; register sign-off before any real pilot bookkeeping |
| Payroll specialist | Future hire; 1 (part-time from Phase 3; implementation Phase 4) | Phase 3 design/registration; Phase 4 STP Phase 2, awards and super |
| Security and DevOps | Future hire; shared with Synapvex | Hosting, backups, monitoring, penetration testing |

Weekly rhythm: Monday planning, daily 15-minute standup, Friday demo to Anujan with adviser-pack updates; accountant review is scheduled at the end rather than required for each weekly change.

## 7. Australian compliance and external processes

These approvals take months, so preparation starts in week 1 in parallel with authorised work. Rules change often: confirm each item against the official source before building it under the provisional-decision model. Current authorisation covers research/documentation only: do not create accounts, send applications/emails or contact providers until Anujan separately authorises those actions.

| Item | Why we need it | Start | Owner |
| --- | --- | --- | --- |
| ATO Digital Service Provider (DSP) Operational Framework | Security controls required from Phase 1; approval required before direct BAS/STP lodgement | Phase 1 preparation; no application sent yet | Product lead |
| SBR / ATO API registration and testing | Machine lodgement of BAS and payroll events | Phase 2 | Tech lead |
| STP Phase 2 product registration | Payroll reporting to the ATO | Phase 3 | Payroll specialist |
| Super clearing and payday super rules | Paying employee super correctly and on time | Phase 3 | Payroll specialist |
| CDR bank feed provider contract (e.g. Basiq) | Live bank transactions from Australian banks | Phase 1 | Product lead |
| Payments provider (e.g. Stripe, PayTo) | Pay-now on invoices | Phase 2 | Tech lead |
| Peppol access point (or partner) | E-invoicing with government and large businesses | Phase 3 | Tech lead |
| Privacy Act compliance and privacy policy | We hold financial and personal data | Phase 1 | Product lead |
| Data hosting in Australia | All customer data, backups, logs and identity in Australian AWS regions; overseas subprocessors need Anujan's written ADR approval | Phase 1 design; no infrastructure applied yet | DevOps |
| ISO 27001 or SOC 2 (later) | Needed to sell to accounting practices and larger firms | Phase 4 | DevOps |
| Accountant adviser end review and register sign-off | Review whole built product and provisional decisions/examples before real pilot bookkeeping | Pack built throughout; adviser signs off before real pilot use | Accountant adviser; Anujan approves provisional decisions before code |

## 8. Working rules for the team

Accounting software is judged on one thing first: the numbers must always be right. These rules protect that. (Full detail is in `AGENTS.md`.)

**Accounting correctness**

- Every transaction is a balanced journal entry: total debits equal total credits, enforced in the database, not just the UI.

- Posted entries are never edited or deleted; corrections are reversing entries. Keep a full audit log of who changed what.

- Money is stored as integer cents (or a decimal type), never floating point.

- Locked periods cannot be changed without an accountant-level permission.

- An automated test checks the trial balance after every test run.

**Code quality**

- Every pull request is reviewed by separate security, code and test agent sessions before Anujan approves the merge; no agent reviews its own work. Anujan, as tech lead, reviews ledger changes.

- Each module ships with unit tests, API tests and one end-to-end test of its main flow.

- Provisional accounting implementation may merge with human review behind a feature flag. Before code, document each rule in `docs/accounting/decisions-register.md` with ID, plain-English rule, numerical examples, official ATO/legislation source, linked "Xero behaviour" (or explicit documentation gap) with deviations explained, and `PROVISIONAL – awaiting adviser review`; Anujan approves each before use. If no clear source exists, record that explicitly, choose the conservative option, add `UNCERTAIN` and report it to Anujan. Never invent a source.

- Keep tax codes, rounding method, posting matrices and account mappings in configuration/data, not hard-coded in the ledger engine. Every provisional rule has golden-file tests.

- Maintain `docs/accounting/adviser-pack/` with the register, readable golden-example tables, demo invoices, credit notes, BAS-style GST summaries and trial balances. The adviser reviews the built product and signs off the register at the end; no real pilot bookkeeping until that hard gate passes.

**Using AI coding tools**

- Give the AI one small task at a time (one endpoint, one screen), never "build the invoicing module".

- Always paste in the relevant data model and this plan's rules with the prompt.

- Treat AI output as a first draft: the developer reads every line, runs the tests, and owns the result.

- Never let AI write accounting rules before their source-backed provisional decision is recorded and approved by Anujan. Use official sources and well-established double-entry practice, flag uncertainty and retain end-review adviser sign-off before real pilot bookkeeping.

- Never paste customer data or secrets into AI tools.

**Security**

- Multi-factor login for all users, role-based permissions, encrypted data at rest and in transit.

- Daily backups with a tested restore, and separate test and production environments.

## 9. Risks and first two weeks

The biggest risk is building too much at once; the plan launches a narrow, correct product first and widens it phase by phase.

| Risk | Effect | Fallback |
| --- | --- | --- |
| Ledger bugs | Wrong numbers, lost trust | Ledger built first by the tech lead, trial-balance tests on every build |
| ATO approvals take longer than planned | No BAS or STP lodgement at launch | Launch with BAS reports for manual lodgement; add direct lodgement later |
| Licence breach from copied code | Forced to open-source or legal action | Clean-room rules, licence register, tech lead approvals |
| Scope creep (payroll, inventory too early) | Nothing ships | Anujan approves development progression and engineering gates; accountant/real-pilot gates constrain real use as described in section 5 |
| Bank feed provider cost | Margin squeeze at low prices | CSV/OFX statement import as a free fallback |
| Accountants slow to switch from Xero | Slow growth | Free accountant hub, Xero data import tool, pilot with friendly practices |

First two weeks:

- [ ] Confirm team members and assign module owners

- [ ] Prepare the decisions register and adviser pack as development proceeds; Anujan will engage an accountant later for finished-product review. Do not wait for an adviser or request development sign-off

- [ ] Clone Bigcapital and Frappe Books locally; record every reference repo's licence

- [ ] Record ledger/accounting decisions with sources/examples before code; Anujan approves provisional decisions and model; adviser reviews at the end before real pilot use

- [ ] After DOCS-000 merge and approved ADRs/briefs, set up repo and CI; document Australian environment design only until infrastructure apply is separately authorised

- [ ] Research DSP security requirements/source evidence; do not send an ATO application or create an account yet

- [ ] Research public bank-feed provider pricing; do not contact providers, create accounts or spend money yet

- [ ] Await Anujan's separate nomination of 3–5 parallel pilot businesses; no real bookkeeping before adviser register sign-off

At the end of each working session, write `docs/status/YYYY-MM-DD.md` (Australia/Sydney date), under one page: PRs opened/merged/waiting/blocked, blocker owners, decisions for Anujan and findings affecting scope/timeline. Then stop and wait.

