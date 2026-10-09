# Bigcapital: invoices, credits and payments

Study: 2026-10-09; agent `/root/ref_bigcapital`; commit `0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0`.
Licence recorded before study in [reference register](../licences/reference-projects.md). AGPL ideas-only; original prose, no source copied.
Static reading only: tests inspected, not executed; findings are bounded to the cited paths. Recommendations require accounting-decision approval.

## Verified observations

- Invoice ledger generation debits receivables, credits item revenue and credits a separate tax-payable account for positive tax amounts; discount and adjustment entries are explicit. [Invoice generator](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/SaleInvoices/ledger/InvoiceGL.ts#L87-L198).
- Creation/delivery events write the invoice ledger only when a delivery timestamp exists. Edits to delivered invoices rewrite its ledger; this differs from our immutable posting rule. [Subscriber](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/SaleInvoices/subscribers/InvoiceGLEntriesSubscriber.ts#L20-L62).
- Customer payment generation debits the deposit account and credits receivables, with additional exchange-difference entries for foreign-currency cases. Phase 1 must reject those non-AUD cases. [Payment generator](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/PaymentReceived/commands/PaymentReceivedGL.ts#L99-L164).
- Credit notes credit receivables and debit the item selling account, plus discount/adjustment entries. The inspected generator's returned list has **no explicit tax-payable reversal**; taxed-credit completeness is unconfirmed and must not be assumed from invoice support. [Credit generator](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/CreditNotes/commands/CreditNoteGL.ts#L80-L171).
- Applying a credit is a separate operation: it locks the credit and referenced invoices, validates remaining amounts, stores allocation links and emits an event within a transaction. The invoice subscriber updates credited amounts. These are allocation limits, not proof that issuing a credit against a paid invoice is prohibited. [Allocation](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/CreditNotesApplyInvoice/commands/CreditNoteApplyToInvoices.service.ts#L51-L130); [Invoice allocation totals](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/CreditNotesApplyInvoice/commands/CreditNoteApplySyncInvoices.service.ts#L24-L36).
- Cash refunds are distinct from allocation: the refund generator debits receivables and credits the selected withdrawal account. [Refund generator](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/CreditNoteRefunds/commands/RefundCreditNoteGLEntries.ts#L61-L113).

## Test and history evidence

- The inspected allocation E2E file creates a credit then checks that the allocation-list endpoint returns success; it does not prove credit ceilings, concurrent allocation or tax reversal. [Allocation test](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/test/credit-notes-apply-invoice.e2e-spec.ts#L61-L76).
- The pinned changelog records discount/adjustment and discount-ledger fixes. Include those combinations in our independent examples. [Changelog](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/CHANGELOG.md#L5-L13).

## Proposed starting position / limits

- Model issue, allocation and cash refund as separate commands and source events; generate each journal through the one ledger gateway in the document transaction.
- Preserve original invoice posting. Credit eligible original value net previous credits even when fully paid; allocate only to available obligations or refund separately, subject to the approved policy.
- Use configurable account mappings and explicit tax reversal from the original tax snapshot. Require examples covering partial payment, full payment, partial credit, discounts and refund.
- **Unconfirmed:** end-to-end taxed-credit behaviour and all subscriber effects; no running application or complete production-safety audit. Do not transplant upstream posting code or schemas.
