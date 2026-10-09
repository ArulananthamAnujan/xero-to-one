# ADR-0001 — Phase 1 scope and delivery gates

Date: 2026-10-09. **Status: PROPOSED — Anujan approval required.** Owner/approver: Anujan. Docs only; no implementation or infrastructure authorisation. This records the approved scope as an architecture boundary; it does not reopen the feature decisions.

## Context and decision proposed

Foundation must deliver an end-to-end AUD bookkeeping workflow before adding bank feeds, lodgement, payroll and automation. Retain the modular monolith in AGENTS.md and the exact phase allocation in BUILD_PLAN.md §5. No microservices, new framework or dependency is selected here.

| Phase 1 included | Boundary |
| --- | --- |
| Ledger, chart of accounts, balanced journal drafts/posting, full reversal, period lock/unlock, trial balance | Single ledger writer; posted history immutable; opening-balance import included |
| Organisations, tenant-local users/memberships/roles | One authorised tenant context per business operation; multi-client hub design only |
| Contacts and ABN Lookup | Resilient lookup adapter, user confirmation, no automated account/provider signup |
| Invoices, credit notes including paid-invoice credits, voids | Versioned posting rules; approval and settlement distinct |
| PDF and email delivery | PDF-001 and EMAIL-001; outbox-backed delivery; overseas providers need written ADR approval |
| Bills, bill approval, file attachments | Quarantine/scan/access controls before files become accessible |
| Manual customer receipts, supplier payments and allocations | Record movements already made; does not initiate external bank payments |
| GST kernel with dated tax codes | Both organisation rounding methods approved provisionally; BAS-style adviser examples are review material, not a shipped BAS report |
| Web app for all above | Original UI/text; no Xero visual copying; accessibility and performance budgets retained |

Phase 2 retains quotes SALES-007/008, recurring SALES-009/010, claims PURCHASES-005/006, OCR/all AI AI-001–003, partner/webhooks PARTNER-001–003, bank feeds/reconciliation, BAS reports/lodgement and launch reporting/provider-payment scope. Multi-currency and Peppol remain Phase 3; payroll specialist design starts part-time in Phase 3, payroll implementation Phase 4. These are boundaries, not permission to start later phases.

Phase 1 and pilot accept AUD only at the posting gateway. Preserve currency columns and NUMERIC(24,10) FX precision for later work; reject non-AUD base or transaction amounts. No multi-currency marketing promise until shipped.

## Gates and sequencing

1. REF-001 is merged and its ledger/posting/rounding directions were approved on 2026-10-09. Record them in the decisions register before any accounting code; all remain provisional and negative rounding remains UNCERTAIN.
2. Anujan separately approves ADR-0001/0002/0003/0004/0006. REPO-001 then requires its approved expanded brief; CI-001 and CI-002 follow by dependency. Never expand more than the next three implementation tasks.
3. Money work depends on approved rounding decisions, approved briefs and CI. Ledger-specific ADR-0005 remains a later dependency; this ADR does not replace it.
4. Every PR receives separate security, code and test reviews before Anujan approves merge. No existing approval is authority to apply cloud infrastructure, create accounts, spend, or contact providers.
5. No real pilot bookkeeping before accountant register sign-off. Synthetic-data development can proceed under approvals; accountant availability is not a per-change gate. Anujan names 3–5 parallel pilots, using their existing books as legal record.
6. DSP security controls apply from Phase 1; if integration approval is delayed at Phase 2 end, manual-BAS reports can support launch while direct lodgement remains disabled.

## Alternatives and consequences

- **Broad Phase 1 with OCR/quotes/API:** rejected because it conflicts with approved deferrals and multiplies acceptance paths before the ledger is proven.
- **Ledger-only Foundation:** rejected because the approved scope includes complete manual sales/purchase settlement and web workflows, not just storage.
- **Approved bounded Foundation:** recommended. Still substantial: current month ranges assumed 6–8 staff, while current capacity is Anujan plus agents. Do not translate agent coding speed into a release-date promise. Re-estimate after REPO/CI/Money PR cycle times, including reviews, evidence and defects.

## Reviewable acceptance boundary

Future tests must demonstrate synthetic opening import → invoice approval → partial receipt → credit/allocation/refund → trial balance, plus bill approval/payment, attachment access, duplicate-command retries, tenant isolation and locked-period rejection. Confirm deferred features are absent from Phase 1 routes and promises. No tests or UI have been implemented by this ADR.

## Sources and decision requested

Project authorities: [AGENTS.md](../../AGENTS.md) §§2–4, 7, 15–16; [BUILD_PLAN.md](../../BUILD_PLAN.md) §§5–9; [REF-001 comparison](../reference-notes/REF-001-comparison.md); Anujan's 9 October approvals. Product scope is a human decision, not a fact requiring third-party endorsement.

**Anujan:** approve this scope/gate record. Provider choices remain for their separate ADRs; no additional phase features or dates are requested.
