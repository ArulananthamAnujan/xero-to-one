# Frappe Books: tax and rounding study

Study: 2026-10-09; session `/root/ref_frappe`; commit `8a4ae666b7805c641d66af53feeab16e00d23c9a`. AGPL ideas-only; original prose, no upstream implementation copied. See [licence register](../licences/reference-projects.md). Static inspection only: no upstream software installed or tests executed.

## Observed design

- Tax templates contain component rates plus invoice and optional payment tax accounts: [tax detail definition](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/schemas/app/TaxDetail.json#L1-L32). The inspected definition uses a numeric Float rate; do not transplant it into our exact-decimal contract.
- The invoice tax routine visits each taxed item and component, adjusts the amount for return sign and pre-tax discount, and multiplies by the rate. Its summary then groups amounts by tax account: [tax calculation](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/Invoice/Invoice.ts#L420-L516).
- Important inspection finding: appending the calculated component is inside the branch where discountAfterTax is false. This is a reason to investigate and test the alternative branch, not a proven end-to-end defect without exercising configuration and surrounding formulas.
- Money construction configures internal and display precision separately: [configuration](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/fyo/index.ts#L151-L164); [defaults](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/fyo/utils/consts.ts#L1-L2) specify 11 and 2 respectively. These are upstream implementation settings, not recommended Australian rounding rules.
- Sales/purchase builders request a round-off entry. The helper posts any nonzero debit/credit difference to the configured round-off account, without a magnitude threshold in that method: [residual helper](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/Transactional/LedgerPosting.ts#L80-L96).

## Lessons and proposed original design

- Preserve a trace from each document line to its dated tax-code version, taxable basis, calculation mode, exact intermediate result and final amount. Account-level aggregation alone must not erase tax-code identity needed for GST reporting.
- Use exact decimal/rational calculations and an approved configurable rounding stage/method. Display precision must never silently determine tax or payment amounts.
- Never use an automatic balancing line to hide a posting defect. Permit only a mathematically explained residual under an approved policy with an explicit bound and recorded reason; otherwise reject the posting.
- Test inclusive/exclusive prices, multiple components/codes sharing an account, discounts before/after tax, half-cent boundaries, negative credits and multi-invoice payment totals. Australian rules must come from official sources and later approved decisions, not this project.

## Regression evidence and limits

- [negative-rate fixture](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/tests/testInvoice.spec.ts#L287-L344) verifies a negative tax reduces the total. It is a project fixture, not an Australian GST treatment recommendation.
- [Issue 770](https://github.com/frappe/books/issues/770) raises currency rounding versus displayed precision; its reporter's proposed Japanese rule is not accepted as legal evidence.
- [Issue 813](https://github.com/frappe/books/issues/813) reports a payment aggregation discrepancy; [v0.35.0](https://github.com/frappe/books/releases/tag/v0.35.0) also records a return-payment outstanding fix. Both motivate tests where displayed totals and allocation totals must agree.
- Unconfirmed: exact Pesa tie-breaking mode at this dependency version, complete tax-inclusive behaviour, and dated Australian GST code support. Dependencies were not installed or separately studied. No Australian tax policy can be inferred from the generic templates.
