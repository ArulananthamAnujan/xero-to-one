# Bigcapital: tax and rounding

Study: 2026-10-09; agent `/root/ref_bigcapital`; commit `0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0`.
Licence recorded before study in [reference register](../licences/reference-projects.md). AGPL ideas-only; original prose, no source copied.
Static reading only: tests inspected, not executed; findings are bounded to the cited paths. Recommendations require accounting-decision approval.

## Verified observations

- Item calculation uses quantity multiplied by rate; inclusive/exclusive amounts and discount values are derived on the item model. Tax helpers take the item amount and rate, and use ordinary JavaScript-number arithmetic without an explicit rounding operation. [Item calculations](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/TransactionItemEntry/models/ItemEntry.ts#L85-L160); [Tax helpers](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/TaxRates/utils.ts#L1-L19).
- In these getters, the tax base is the quantity/rate amount, while discounts are subtracted separately. This is a path-specific observation, not an Australian GST rule or a claim about every creation transformer.
- Tax-rate records, item rate snapshots and tax transaction records exist. Their initial migration does not establish the effective-dated code/version policy required by our handbook. [Tax migration](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/database/tenant/migrations/20230810191606_create_tax_rates.ts#L1-L48).
- Invoice totals combine subtotal, discount, adjustment and tax according to inclusive/exclusive mode. This separates components but does not itself define line-versus-document rounding. [Invoice totals](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/SaleInvoices/models/SaleInvoice.ts#L210-L229).
- A later migration stores document tax totals at two decimal places and increases their magnitude capacity; the initial ledger migration used three decimal places. Persistence precision alone is not a consistent calculation policy. [Tax precision](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/database/tenant/migrations/20240801130829_change_tax_amount_withheld_column_precision_in_bills_and_sales_invoices_tables.ts#L1-L11); see [ledger note](bigcapital-ledger.md).

## Test and history evidence

- Manual-journal tests inspect balanced whole-number fixtures and FX multiplication, not GST half-cent boundaries. No directly applicable Australian GST rounding golden test was confirmed in the inspected paths. [Tests](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/ManualJournals/commands/ManualJournalGL.spec.ts#L39-L79).
- Decimal balancing and discount-posting fixes in the pinned changelog identify useful edge-case classes; they do not establish a rounding standard. [Changelog](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/CHANGELOG.md#L5-L29).
- Also read [release v0.15.43](https://github.com/bigcapitalhq/bigcapital/releases/tag/v0.15.43): its listed mail/report presentation fixes do not validate ledger/tax correctness. Release context is distinct from the pinned code.

## Proposed starting position / limits

- Use exact decimal/rational intermediate calculations and integer-minor-unit posted values through our Money package; reject floating-point money at all boundaries.
- Configure calculation stage, tie handling, discount tax base and permitted residual account. Persist original tax code/version, inputs and rounded outputs so credits reverse the original calculation.
- Require approved official-source examples for inclusive/exclusive prices, mixed tax classifications, discounts, half-cent ties, negative adjustments and a partial credit followed by final credit.
- **Unconfirmed:** Bigcapital's complete final rounding policy, effective dating across all migrations and Australian GST suitability. Do not use its arithmetic as tax authority; ATO/legislation evidence and Anujan's provisional-decision approval remain prerequisites.
