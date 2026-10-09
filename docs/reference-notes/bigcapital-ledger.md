# Bigcapital: ledger and integrity

Study: 2026-10-09; agent `/root/ref_bigcapital`; commit `0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0`.
Licence recorded before study in [reference register](../licences/reference-projects.md). AGPL ideas-only; original prose, no source copied.
Static reading only: tests inspected, not executed; findings are bounded to the cited paths. Recommendations require accounting-decision approval.

## Verified observations

- The ledger stores account transaction rows with debit/credit, account, source reference, contact, date and ordering metadata; manual journals also have their own document/entry tables. This is a source-document-to-ledger projection, not evidence for our proposed header/line schema. [Initial transaction migration](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/database/tenant/migrations/20200104232647_create_accounts_transactions_table.ts#L1-L31).
- The manual-journal validator sums each side, rounds each sum to two places, rejects nonpositive totals and then compares them. That is an application check and can conceal sub-cent differences; it is not exact persisted-line equality. [Validator](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/ManualJournals/commands/CommandManualJournalValidators.service.ts#L29-L49).
- The common storage method saves rows and updates account/contact balances using the supplied transaction. The inspected entry storage drops zero entries and inserts rows; neither inspected method independently asserts a balanced batch. [Storage](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/Ledger/LedgerStorage.service.ts#L32-L55); [Entry storage](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/Ledger/LedgerEntriesStorage.service.ts#L29-L75).
- Posted invoice projections are mutable: invoice edit events call a rewrite that deletes the prior source-reference rows and writes replacement rows. Its reversal helper reverses balance effects while deleting rows, rather than preserving an opposite journal. [Rewrite](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/SaleInvoices/ledger/InvoiceGLEntries.ts#L64-L94); [Delete by reference](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/Ledger/LedgerStorage.service.ts#L96-L117).

## Test and history evidence

- Inspected manual-journal tests cover same-currency amounts, exchange multiplication and equal aggregate totals in a balanced fixture. They do not establish database enforcement or concurrent-write safety. [Tests](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/packages/server/src/modules/ManualJournals/commands/ManualJournalGL.spec.ts#L39-L89).
- The pinned changelog records decimal-balance fixes and subsequent decimal-entry handling; these are reasons to test exact minor-unit equality, not proof those historical bugs remain. [Changelog](https://github.com/bigcapitalhq/bigcapital/blob/0ca638f9c3c3b0847e6d27bb2735e836d5a79bb0/CHANGELOG.md#L19-L29).
- [Issue 1445](https://github.com/bigcapitalhq/bigcapital/issues/1445) reports unbounded reconciliation candidate loading. Treat as a report, not a reproduced benchmark; use bounded queries in our design.

## Proposed starting position / limits

- Keep the useful separation between business documents, journal generation and balance projections, expressed through our existing module boundaries.
- Do not adopt delete-and-rewrite for posted records. Retain immutable posted headers/lines, linked compensating entries, exact database balance enforcement and the handbook's shared lock order.
- Cached balances must be reconstructible from immutable lines. Test duplicate commands, concurrent edits/locks and direct invalid SQL independently of UI validation.
- **Unconfirmed:** whether other migrations or runtime controls add further safeguards; this study does not claim every Bigcapital posting route can persist imbalance. No database instance was run.
