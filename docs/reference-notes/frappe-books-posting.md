# Frappe Books: document posting study

Study: 2026-10-09; session `/root/ref_frappe`; commit `8a4ae666b7805c641d66af53feeab16e00d23c9a`. AGPL ideas-only; original prose, no upstream implementation copied. See [licence register](../licences/reference-projects.md). Static inspection only: no upstream software installed or tests executed.

## Observed design

- Sales submission normally debits the invoice control account and credits item accounts and tax accounts. Return invoices swap the sides; configured discounts and a round-off entry complete the posting: [sales posting](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/SalesInvoice/SalesInvoice.ts#L25-L95).
- Purchase invoices use the opposite direction: credit the invoice control account, debit item and tax accounts, and reverse those directions for returns: [purchase posting](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/PurchaseInvoice/PurchaseInvoice.ts#L54-L103).
- Credits are represented through return invoices linked to the original, rather than a separate credit-note model in these paths. The outstanding calculation explicitly handles a fully paid original: [return outstanding](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/Invoice/Invoice.ts#L1217-L1243).
- Payment postings debit the destination account and credit the source account; the receive/pay mode determines which business accounts occupy those positions. Optional tax-transfer and write-off postings are separate adjustments: [payment posting](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/Payment/Payment.ts#L331-L392).
- Payment allocations are child references; the amount updater sums references, while validation checks references and totals: [allocation validation](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/Payment/Payment.ts#L95-L121).

## Lessons and proposed original design

- Keep invoice recognition, credit issuance, credit allocation and cash refund as explicit separate domain events. Configure posting matrices outside the ledger engine; preserve source and allocation links.
- Use separate credit documents for our Phase 1 workflow. A paid invoice remains creditable subject to original eligible value less prior credits; do not constrain credit issuance by outstanding cash balance.
- Proposed synthetic no-tax fixture: invoice 100 debits receivables 100 and credits revenue 100; receipt 100 debits bank 100 and credits receivables 100; credit 20 debits revenue 20 and credits receivables 20; refund 20 debits receivables 20 and credits bank 20. These are proposed double-entry examples, not Australian tax advice or approved posting rules.
- Allocation should not recognise revenue a second time. Test partial allocations, multiple invoices, over-credit rejection, cancellation dependencies and receipt reversal separately.

## Regression evidence and limits

- [payment regression](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/tests/testPayment.spec.ts#L37-L98) constructs two invoices and verifies their combined payment passes amount validation; test inspected, not run.
- [return tests](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/tests/testInvoice.spec.ts#L65-L218) covers payment before a sales return, expected return ledger sides, further returns and a return payment.
- [v0.35.0 changelog](https://github.com/frappe/books/releases/tag/v0.35.0) records a fix for outstanding balances on return payments. Make the credit/refund lifecycle a first-class regression suite.
- [Issue 813](https://github.com/frappe/books/issues/813) reports a one-cent discrepancy when combining purchase payments in an older version. It motivates reconciliation assertions; it does not establish that the pinned commit still has that bug.
- Unconfirmed: every return/partial-payment/discount permutation and transactional protection under concurrent users. Our cloud design requires independent proofs, not desktop workflow parity alone.
