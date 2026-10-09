# Accounting decisions register

Owner and provisional approver: Anujan. Recorded 2026-10-09 after his explicit approval of REF-001 directions and merge of [PR #4](https://github.com/ArulananthamAnujan/xero-to-one/pull/4). All entries remain **PROVISIONAL – awaiting adviser review**. Provisional direction approval is recorded below; this does not authorise unapproved task briefs, unselected ADRs or real bookkeeping. No accounting code exists. Feature flags and accountant sign-off before real pilot use remain mandatory.

## ACCT-001 — ledger integrity and reversals

- **Status:** PROVISIONAL – awaiting adviser review. Direction approved by Anujan, 2026-10-09.
- **Rule:** use our own journal model. Every committed ledger draft has at least two lines and balances; incomplete edits stay in document drafts. Posted amounts/lines never change or disappear. Full reversals use the exact saved amounts and opposite sides, linked to the original. Keep permanent tenant/source-event uniqueness even after response-cache expiry. A changed retry payload rejects; a duplicate event cannot generate another journal.
- **Basis / official source:** established double-entry practice plus Anujan's engineering decisions and AGENTS.md §§3–5; [ATO record-keeping guidance](https://www.ato.gov.au/businesses-and-organisations/preparing-lodging-and-paying/record-keeping-for-business) is the official context to collect/confirm for record retention. It is not claimed to prescribe journal tables, immutable SQL rows, lock order or idempotency. Those are our technical safeguards, not invented ATO mandates.
- **Xero behaviour:** [Journals](https://developer.xero.com/documentation/api/accounting/journals/) documents source references and debit/credit signs; [Types](https://developer.xero.com/documentation/api/accounting/types/) includes posted and voided manual journals. Xero's physical storage guarantees are not documented here; our immutable model is an explicit design choice.
- **Reference projects:** [Bigcapital ledger](../reference-notes/bigcapital-ledger.md) separates source documents from ledger projections but rewrites invoice projections; [Frappe ledger](../reference-notes/frappe-books-ledger.md) balances with Money and creates reversal rows while allowing other history mutations. Retain source linkage, not their mutation semantics.
- **Worked examples:** Dr receivable 110.00 / Cr revenue 100.00 / Cr tax payable 10.00 balances at 110.00. Reversal is Dr revenue 100.00 / Dr tax payable 10.00 / Cr receivable 110.00; all account totals net zero even if the tax code or rounding policy later changes. A draft Dr 110.00 / Cr 109.99 rejects. Retry event `demo-invoice-001:approve:v1` twice produces one 110.00 journal, including after the replay cache expires.
- **Planned golden/property cases:** `balanced-draft`, `reject-one-cent-imbalance`, `exact-saved-reversal`, `late-source-event-retry`, `same-key-different-payload`. Tests not implemented yet.

## ACCT-002 — posting matrix, credits, allocations and refunds

- **Status:** PROVISIONAL – awaiting adviser review. Direction approved by Anujan, 2026-10-09.
- **Rule:** version and configure the posting matrix and account mappings. Invoice recognition, receipt, credit creation, credit allocation and refund are distinct events. Credit issuance is limited by original eligible value less prior credits, not unpaid balance. Within the same receivable account, AUD currency and tenant/contact, allocation changes settlement records without a new journal. A refund does create a separate cash journal. Period locks and permissions still apply.
- **Basis / official source:** established double-entry practice; [ATO tax invoices](https://www.ato.gov.au/businesses-and-organisations/gst-excise-and-indirect-taxes/gst/tax-invoices) is tax-document context, not an authority for our account matrix. Source-backed adjustment-note eligibility and timing must be researched before credit-note implementation; this entry approves event separation, not every Australian adjustment treatment.
- **Xero behaviour:** [Invoices](https://developer.xero.com/documentation/api/accounting/invoices) distinguishes approval from settlement; [Credit Notes](https://developer.xero.com/documentation/api/accounting/creditnotes/) separates creation and allocation to outstanding invoices; [Payments](https://developer.xero.com/documentation/api/accounting/payments) covers refunds and payment reversal. No-new-journal allocation is our approved same-control-account design; do not assert Xero's hidden journal implementation.
- **Reference projects:** [Bigcapital posting](../reference-notes/bigcapital-posting.md) separates allocation/refund; [Frappe posting](../reference-notes/frappe-books-posting.md) shows return documents and payment direction. Taxed-credit completeness in Bigcapital remains unconfirmed.
- **Worked examples:** see the table below. Amount splits are supplied synthetic fixtures, not assertions of tax eligibility. Every journal event balances; allocation is an audited non-journal event.

| Example | Debit | Credit | Result |
| --- | --- | --- | --- |
| Invoice net 100.00, supplied tax 10.00 | Receivable 110.00 | Revenue 100.00; tax payable 10.00 | Due 110.00 |
| Receipt 60.00 | Bank 60.00 | Receivable 60.00 | Due 50.00; no revenue recognised again |
| Credit net 20.00, supplied tax 2.00 | Revenue 20.00; tax payable 2.00 | Receivable 22.00 | Credit available 22.00 |
| Allocate that credit to same-control invoice | None | None | Due 28.00; credit available 0.00 |
| Alternative: credit 22.00 after invoice fully paid | Revenue 20.00; tax payable 2.00 | Receivable 22.00 | Invoice remains paid; credit available 22.00 |
| Refund that paid-invoice credit | Receivable 22.00 | Bank 22.00 | Credit available 0.00 |
| Bill, supplied cost 50.00 and recoverable tax 5.00 | Expense/asset 50.00; recoverable tax 5.00 | Payable 55.00 | Due 55.00; recoverability needs separate evidence |
| Supplier payment | Payable 55.00 | Bank 55.00 | Due 0.00 |

- **Planned cases:** `receipt-no-duplicate-revenue`, `same-control-allocation-no-journal`, `paid-invoice-credit-refund`, `credit-cap-original-less-prior`, `allocation-over-open-due-rejected`, `cross-tenant-allocation-denied`.
- **Not yet decided:** different-control-account allocation, partial-credit tax distribution, supplier tax overrides, paid-document reclassification and cash-basis reporting attribution. Do not silently extrapolate these fixtures.

## ACCT-003 — organisation GST rounding policy

- **Status:** PROVISIONAL – awaiting adviser review. Default and alternative approved by Anujan, 2026-10-09. Negative tie handling additionally **UNCERTAIN**.
- **Rule:** default `TAXABLE_SALE_CENTS`: calculate each sale/line's tax, round to cents, then sum; positive half-cent rounds up. Offer `TOTAL_INVOICE`: sum unrounded tax for the invoice, then round once to cents. Store the choice per organisation and snapshot policy/version on issued documents. Settings changes must not recalculate issued documents. Each line represents a taxable sale for this default; more complex supply grouping needs a separate decision.
- **Negative rule:** use half away from zero (for example -0.005 becomes -0.01), as expressly approved by Anujan to mirror positive ties. This is **UNCERTAIN**: the retrieved ATO text does not establish signed credit-note rounding. Full credits/reversals negate saved original amounts and never recalculate, even after an organisation changes method.
- **Official evidence, rechecked 2026-10-09:** [ATO tax invoices](https://www.ato.gov.au/businesses-and-organisations/gst-excise-and-indirect-taxes/gst/tax-invoices) and its [official PDF](https://www.ato.gov.au/api/public/content/0-1e92db95-a75c-4f4e-a3d4-39f43b1a3b25). Direct retrieval still returned 403. Official indexed PDF text confirms individual-sale calculation, rounding to system precision before summation, final cents rounding, and that customer/supplier methods may differ. Indexed ATO text names the total-invoice method and a GST-exclusive aggregation alternative. This supports the positive-method direction; it is not a complete direct-page capture.
- **Wording check:** the indexed total-invoice sentence is less explicit about unrounded aggregation than Anujan's summary. No contradictory method was found, but retain this limitation for adviser/source verification. The exact algorithm above is the product lead's approved provisional interpretation. [Oreon](https://www.oreon.com.au/?p=3481), supplied by Anujan, could not be retrieved and is not used as legal authority.
- **Xero behaviour:** [Tax and the API](https://developer.xero.com/documentation/guides/how-to-guides/tax-in-xero/) documents per-line rounding to two decimals. That matches the chosen default. Complete negative tie behaviour was not confirmed; offering total-invoice rounding is an explicit product option, not a claim of Xero parity for both modes.
- **Reference projects:** [Bigcapital tax](../reference-notes/bigcapital-tax-rounding.md) has numeric helpers without a confirmed end-to-end rounding policy; [Frappe tax](../reference-notes/frappe-books-tax-rounding.md) separates internal/display precision and includes residual handling. Neither supplies the Australian rule or our tie policy.

| Synthetic input (supplied test rate 10%) | Taxable-sale cents | Total-invoice | Significance |
| --- | --- | --- | --- |
| Net 0.05 + 0.05; raw tax 0.005 + 0.005 | 0.01 + 0.01 = 0.02; gross 0.12 | 0.010 rounds to 0.01; gross 0.11 | Required one-cent divergence; both remain exactly balanced |
| Net 45.45 + 45.45; raw tax 4.545 + 4.545 | 4.55 + 4.55 = 9.10; gross 100.00 | 9.090 rounds to 9.09; gross 99.99 | REF-001 compatibility example |
| Negative net -0.05 -0.05; raw tax -0.005 -0.005 | -0.01 -0.01 = -0.02 | -0.010 rounds to -0.01 | Provisional signed mirror, UNCERTAIN |
| Full credit of first row originally issued under taxable-sale mode; org now total-invoice | Reverse saved net 0.10 and tax 0.02, total 0.12 | Current setting ignored | No loss of saved one-cent difference |

- **Scope of these rounding examples:** all multiplication examples are tax-exclusive. The tax-inclusive calculation order is not approved by this entry: do not apply net-times-rate directly to a gross price. Xero documents rounded gross less rounded net; a separate inclusive-policy decision with fixtures is required before implementing that path.
- **Implementation boundary:** exact decimal/rational intermediate arithmetic; AUD posted cents; no binary floating-point Money. Do not sum rounded line tax under TOTAL_INVOICE. Header tax must reconcile to tax-code/account totals; deterministic residual distribution across lines/codes is not yet approved and must be decided before that path is implemented. No unrestricted rounding-account plug.
- **Planned cases:** `taxable-sale-half-cent`, `total-invoice-half-cent`, `negative-tie-away-from-zero`, `saved-full-credit-after-policy-change`, `policy-snapshot-stable`, `total-tax-breakdown-reconciles`. Golden fixtures must be versioned and reviewed; none are executable yet.

## Outstanding initial decisions

The original Part E also requires a dedicated period-lock decision with numerical/date examples before that code. AGENTS.md lock/unlock permissions and deterministic lock order remain mandatory, but this register does not yet supply its full approved rule/fixtures. Tax-inclusive calculation, partial credits and total-invoice line/code residual distribution also need explicit follow-up decisions. These three entries are the first approved directions, not a complete accounting specification.

## Review and release gate

Keep every implementation using these rules feature-flagged. The adviser pack includes [readable examples](adviser-pack/initial-decisions.md). Independent tests must prove the eventual code follows the approved fixtures. Accounting code still needs approved briefs; real pilot bookkeeping still requires accountant review and signature of this register. Source limitations and UNCERTAIN negative rounding remain visible until resolved, never relabelled as adviser-approved.
