# Accounting Platform Build Plan

Version: 9 Oct 2026 · Owner: Anujan (product lead)

## 1. Goal and how we beat Xero

We build an Australian-first cloud accounting platform that matches Xero's core and wins on price, AI automation and local compliance. We do not merge the open-source projects; we study them and write our own code.

Where we beat Xero:

- **Price:** no invoice or bill caps on the entry plan, and multi-currency on every plan (Xero restricts both to higher tiers).

- **AI bookkeeping:** auto-categorise bank transactions, read receipts and bills, and suggest reconciliations with a confidence score.

- **Australia built in, not added on:** BAS, GST, STP Phase 2 payroll and super handled natively, not through paid add-ons.

- **Community focus:** Tamil, Hindi and Sinhala invoice templates and language support for the South Asian small-business market we already serve.

- **Accountant tools:** one dashboard for practices to manage many clients, with bulk BAS lodgement.

- **Open API from day one:** so other apps (POS, Shopify, Square) connect easily.

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

              (receipts, PDFs) | Job queue (Redis)             AI model API (OCR, LLM)

```

- **Backend:** Node.js with TypeScript (NestJS). Same language as Bigcapital, so the team can study it easily.

- **Web:** React with TypeScript. **Mobile:** Flutter, which the team already uses.

- **Database:** PostgreSQL, every row tagged with its organisation, row-level security on.

- **Background jobs:** Redis queue for bank feed syncs, ATO lodgements and emails.

- **Hosting:** AWS Sydney region, separate test and production.

- **AI:** an LLM API plus OCR for receipts; every AI suggestion needs a user's approval before it posts.

## 4. Module map

Sixteen modules, built on one shared ledger: every module posts journal entries to the ledger and never stores balances of its own.

| Module | What it does | Reference project | Australian requirement | Phase |
| --- | --- | --- | --- | --- |
| Core ledger | Chart of accounts, journals, trial balance, period lock | Frappe Books, Beancount | AU standard chart of accounts template | 1 |
| Contacts | Customers, suppliers, ABN lookup | Bigcapital | ABN validation via ABR lookup | 1 |
| Invoicing | Quotes, invoices, credit notes, recurring, templates | Invoice Ninja, Bigcapital | Tax invoice rules (ABN, GST shown) | 1 |
| Bills and expenses | Bills, receipts, expense claims, AI receipt reading | Bigcapital, Crater | GST credits on purchases | 1 |
| Banking | Bank feeds, statement import, reconciliation | Odoo, GnuCash | CDR bank feeds via a provider (e.g. Basiq) | 2 |
| Payments | Pay-now links on invoices, card and PayTo | Invoice Ninja | PayTo and BPAY options | 2 |
| GST and BAS | GST codes, BAS report, lodgement | ERPNext (tax engine) | ATO lodgement via SBR, DSP framework | 2 |
| Reporting | P&L, balance sheet, cash flow, aged receivables/payables | Bigcapital | AU report formats | 2 |
| Inventory | Items, stock levels, cost of goods | Bigcapital | None specific | 3 |
| Fixed assets | Asset register, depreciation | ERPNext | ATO depreciation methods, instant asset write-off | 3 |
| Projects | Time tracking, job costing | ERPNext | None specific | 3 |
| Payroll | Pay runs, leave, payslips, super | ERPNext | STP Phase 2, Fair Work awards, super clearing | 4 |
| Multi-currency | Foreign invoices, revaluation | ERPNext, GnuCash | RBA exchange rates | 3 |
| E-invoicing | Send and receive Peppol invoices | None (build from spec) | Peppol access point accreditation | 4 |
| AI assistant | Categorise, reconcile, answer questions about the books | None (our differentiator) | Explainable suggestions, human approval | 2 onward |
| Accountant hub | Multi-client dashboard, bulk BAS | Odoo | Tax agent workflows | 4 |

## 5. Phased roadmap

The paid MVP launches at month 8 with invoicing, bills, bank feeds, GST/BAS and AI categorising; payroll comes last because it carries the most compliance risk.

| Phase | Months | Name | Scope | Gate to pass before next phase |
| --- | --- | --- | --- | --- |
| 1 | 1–4 | Foundation | Core ledger, contacts and ABN lookup, invoicing, bills and expenses, web app | Pilots run their books for 1 month; trial balance tests pass |
| 2 | 5–8 | Launch MVP | Bank feeds, reconciliation, GST and BAS, reports, payments, AI categorising | Adviser signs off BAS; DSP approval in place |
| 3 | 9–13 | Grow | Inventory, fixed assets, projects, multi-currency, mobile app, Peppol e-invoicing | Paying customers stable; payroll design approved |
| 4 | 14–18 | Payroll and practices | STP Phase 2 payroll, super, leave, awards, accountant hub, Xero import tool, ISO 27001 start | — |

A phase starts only when the previous gate is passed. The month ranges assume the core team below and are estimates to revisit at each gate.

## 6. Team structure

A core team of 6 to 8 people can ship Phase 1 and 2; payroll and e-invoicing need one more specialist each. Each module has one owner who is accountable for it end to end.

| Role | Count | Owns |
| --- | --- | --- |
| Product lead (Anujan) | 1 | Scope, priorities, compliance applications, final sign-off |
| Tech lead / architect | 1 | Architecture, ledger engine, code review, licence approvals |
| Backend developers | 2 | Contacts, invoicing, bills, banking, GST/BAS, reporting APIs |
| Frontend developer | 1 | Web app UI, design system, invoice templates |
| Mobile developer | 1 (from Phase 3) | iOS/Android app: receipts, invoices, approvals |
| AI/ML engineer | 1 | Categorisation, receipt reading, reconciliation suggestions |
| QA and accounting tester | 1 | Test plans, accounting correctness, regression suite |
| Accountant adviser (CPA/CA, part-time) | 1 | Reviews every module's accounting logic and AU tax rules |
| Payroll specialist | 1 (Phase 4) | STP Phase 2, awards, super |
| Security and DevOps | shared with Synapvex | Hosting, backups, monitoring, penetration testing |

Weekly rhythm: Monday planning, daily 15-minute standup, Friday demo to the accountant adviser.

## 7. Australian compliance and external processes

These approvals take months, so they start in week 1, in parallel with development. Rules change often: confirm each item against the official source before building it.

| Item | Why we need it | Start | Owner |
| --- | --- | --- | --- |
| ATO Digital Service Provider (DSP) Operational Framework | Required before our software can lodge BAS or STP with the ATO | Phase 1 | Product lead |
| SBR / ATO API registration and testing | Machine lodgement of BAS and payroll events | Phase 2 | Tech lead |
| STP Phase 2 product registration | Payroll reporting to the ATO | Phase 3 | Payroll specialist |
| Super clearing and payday super rules | Paying employee super correctly and on time | Phase 3 | Payroll specialist |
| CDR bank feed provider contract (e.g. Basiq) | Live bank transactions from Australian banks | Phase 1 | Product lead |
| Payments provider (e.g. Stripe, PayTo) | Pay-now on invoices | Phase 2 | Tech lead |
| Peppol access point (or partner) | E-invoicing with government and large businesses | Phase 3 | Tech lead |
| Privacy Act compliance and privacy policy | We hold financial and personal data | Phase 1 | Product lead |
| Data hosting in Australia | Customer and accountant trust; some sectors require it | Phase 1 | DevOps |
| ISO 27001 or SOC 2 (later) | Needed to sell to accounting practices and larger firms | Phase 4 | DevOps |
| Accountant adviser sign-off per module | Accounting correctness | Every phase | Accountant adviser |

## 8. Working rules for the team

Accounting software is judged on one thing first: the numbers must always be right. These rules protect that. (Full detail is in `AGENTS.md`.)

**Accounting correctness**

- Every transaction is a balanced journal entry: total debits equal total credits, enforced in the database, not just the UI.

- Posted entries are never edited or deleted; corrections are reversing entries. Keep a full audit log of who changed what.

- Money is stored as integer cents (or a decimal type), never floating point.

- Locked periods cannot be changed without an accountant-level permission.

- An automated test checks the trial balance after every test run.

**Code quality**

- All work goes through pull requests with at least one reviewer; the tech lead reviews anything touching the ledger.

- Each module ships with unit tests, API tests and one end-to-end test of its main flow.

- A module is "done" only when the accountant adviser has signed off its test scenarios.

**Using AI coding tools**

- Give the AI one small task at a time (one endpoint, one screen), never "build the invoicing module".

- Always paste in the relevant data model and this plan's rules with the prompt.

- Treat AI output as a first draft: the developer reads every line, runs the tests, and owns the result.

- Never let AI write tax calculations or payroll rules without checking them against the ATO source and the accountant adviser.

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
| Scope creep (payroll, inventory too early) | Nothing ships | Phase gates: next phase starts only when the previous one passes |
| Bank feed provider cost | Margin squeeze at low prices | CSV/OFX statement import as a free fallback |
| Accountants slow to switch from Xero | Slow growth | Free accountant hub, Xero data import tool, pilot with friendly practices |

First two weeks:

- [ ] Confirm team members and assign module owners

- [ ] Engage a part-time CPA/CA accountant adviser

- [ ] Clone Bigcapital and Frappe Books locally; record every reference repo's licence

- [ ] Tech lead writes the ledger data model and gets adviser sign-off

- [ ] Set up repo, CI pipeline, test and production environments in Australia

- [ ] Start the ATO DSP Operational Framework application

- [ ] Contact bank feed providers and compare pricing

- [ ] Pick 3 to 5 pilot businesses (e.g. from A2Y and Synapvex clients) for Phase 1 testing
