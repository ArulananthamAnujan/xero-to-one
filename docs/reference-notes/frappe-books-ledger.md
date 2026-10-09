# Frappe Books: ledger design study

Study: 2026-10-09; session `/root/ref_frappe`; commit `8a4ae666b7805c641d66af53feeab16e00d23c9a`. AGPL ideas-only; original prose, no upstream implementation copied. See [licence register](../licences/reference-projects.md). Static inspection only: no upstream software installed or tests executed.

## Observed design

- Document schemas distinguish manual journals and their account rows from the resulting accounting ledger rows. Ledger rows carry account, party, date, debit/credit, source-document identity and reversal linkage: [ledger schema](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/schemas/app/AccountingLedgerEntry.json#L1-L98) and [journal builder](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/JournalEntry/JournalEntry.ts#L24-L44).
- A posting accumulator consolidates amounts by account and debit/credit side. It checks exact Money equality before saving each ledger row; transactional document validation also invokes that check. Evidence: [posting](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/Transactional/LedgerPosting.ts#L40-L62) and [balance/save](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/Transactional/LedgerPosting.ts#L137-L172).
- The balance check is application logic. The inspected storage backend uses SQLite: [database](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/backend/database/core.ts#L48-L65). This study does not establish a database-level balance constraint or atomicity across the full document-and-ledger operation.
- Submission creates postings; cancellation asks existing source-linked rows to reverse. A reversal swaps amounts into a new linked row but also updates the original row's reverted marker: [reversal](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/AccountingLedgerEntry/AccountingLedgerEntry.ts#L16-L41).
- This is not strict append-only immutability. Submitted documents advertise no editing, while cancelled documents can be deleted; the transactional deletion hook deletes associated ledger rows. Evidence: [document lifecycle](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/fyo/model/doc.ts#L140-L183) and [delete hook](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/Transactional/Transactional.ts#L73-L97).

## Lessons and proposed original design

- Keep a single accounting gateway and explicit source linkage, but implement our own tenant-bound journal header/line design and PostgreSQL constraints already required by AGENTS.md.
- Enforce balanced committed drafts and immutable posted records in the database. Retain originals unchanged and append a separately identified reversal; do not adopt deletion of cancelled ledger history.
- Make source document, ledger, audit and outbox writes one transaction. Prove crash rollback and concurrent retries with real PostgreSQL tests; the inspected sequential save loop is not evidence for these guarantees.
- Keep document draft edits separate from accounting records. No new accounting policy is approved by this study.

## Regression evidence and limits

- [invoice tests](https://github.com/frappe/books/blob/8a4ae666b7805c641d66af53feeab16e00d23c9a/models/baseModels/tests/testInvoice.spec.ts#L95-L155) inspect return debit/credit sides and prohibit cancelling an invoice with a linked return; these are useful scenarios, not proof of database invariants.
- [v0.36.0 release notes](https://github.com/frappe/books/releases/tag/v0.36.0) add party-ledger reporting. Release notes describe product evolution, not proof that balances survive arbitrary writes.
- Unconfirmed: comprehensive direct-write protection, transaction/crash guarantees and period-lock concurrency. No blanket claim that upstream lacks them is made.
