# REF-001 — ledger, posting and rounding comparison

Prepared for Anujan, 9 October 2026. **Recommendations for review, not approved accounting rules or implementation authority.** No production code, Xero login/API use, or upstream software execution. Six original design notes below contain commit-pinned evidence; the [licence register](../licences/reference-projects.md) records provenance. Static inspection does not prove runtime correctness or exhaust every code path.

## Recommended starting position

1. **Ledger:** retain our own PostgreSQL journal header/line model, exact amounts, balanced committed drafts, immutable posted rows and linked reversals. Learn from the projects' central posting boundaries and document provenance; do not transplant their schemas or mutation behaviour. Database enforcement, tenant isolation and concurrency remain our independent responsibilities.
2. **Posting:** use explicit document commands and a versioned, configurable posting matrix. Approval creates accounting entries; settlement is separate from revenue recognition. Treat credit creation, credit allocation and refund as distinct operations. A fully paid invoice can justify a credit without any remaining amount to allocate against it.
3. **Tax/rounding:** propose exact decimal calculation with a configurable, versioned policy and dated tax codes. Use Xero's documented per-line calculation as the compatibility target, subject to official Australian requirements. Do not adopt either project's arithmetic as Australian GST authority. Negative ties, partial-credit residuals and supplier-total adjustments need explicit decisions before code.

## Evidence and comparison

| Topic | Bigcapital | Frappe Books | Xero public documentation | Our starting position |
| --- | --- | --- | --- | --- |
| Journal model and protection | See [ledger note](bigcapital-ledger.md): source-linked account transactions; manual validation compares rounded totals; inspected shared storage does not itself assert batch balance. Invoice edits delete/rewrite ledger rows. | See [ledger note](frappe-books-ledger.md): central Money balance check; cancellation adds inverse rows but changes original metadata, and cancelled-document deletion can remove ledger history. | [X1] exposes source-linked journal lines and signed debit/credit amounts. [X2] lists draft/posted/deleted-draft/voided-posted manual-journal states. Public interfaces do not reveal internal tables, triggers or locks. | Keep source-event uniqueness permanently; enforce balance at commit and posted-row immutability in PostgreSQL. Do not claim this reproduces Xero's internal implementation. |
| Invoice, credit and payment posting | [Posting note](bigcapital-posting.md) traces invoice, credit and receipt adapters. Taxed-credit completeness must not be assumed from an invoice path. | [Posting note](frappe-books-posting.md) traces invoice submission, returns and payments. Its return model is an implementation idea, not automatically our credit-note contract. | [X3] separates unposted drafts/submitted invoices from approved invoices that generate journals. [X4] separates credit allocation from credit creation; [X5] treats refunds as payments and supports payment reversal. | Atomic document transition, posting, audit and outbox. Separate allocation records with tenant/contact/currency checks. Never recognise revenue again when receiving cash against an existing invoice. |
| Tax and rounding | [Tax note](bigcapital-tax-rounding.md) traces tax helpers and numeric operations; their existence is not proof of an explicit AU rounding policy. | [Tax note](frappe-books-tax-rounding.md) separates internal/display precision and aggregates tax by account. Its residual helper balances any nonzero difference without a bound in that method; do not adopt that safeguard level. | [X6] says taxes are calculated/rounded per line to cents; inclusive tax is the rounded gross line less the rounded exclusive amount. [X7] rejects a generic separate tax-only line workaround for AU customers. | Exact decimal intermediates; explicit rounding stages; preserve calculation inputs, policy version, taxable basis and amounts on issued documents. No unexplained balancing plug. |

### Ledger recommendation and deliberate differences

The existing handbook already requires at least two lines and equality of debits/credits for every committed ledger draft. Incomplete editing belongs in document drafts. Neither a project's draft model nor Xero's document states is permission to relax this invariant.

Posting should acquire period then journal locks in the handbook's deterministic order, revalidate state, and commit document/posting/audit/outbox together. Only the ledger module writes its tables. Permanent tenant/source-event uniqueness prevents a retry after response-cache expiry from posting twice. These are our engineering proposals/requirements, not observations of Xero's private architecture.

Full reversal should negate stored original amounts and link to the original entry, without recalculating historical tax using today's configuration. Reclassification or document edits affecting posted amounts require an explicit compensating workflow. Xero permits some changes to approved/paid documents [X3]; matching the workflow does not require editing our posted journal rows. Exact user-facing amendment scope remains for the decisions register.

### Posting examples for review

Illustrative AUD fixtures below assume already-approved document splits; they do not determine tax eligibility or a tax rate. Account names are logical roles resolved through configurable mappings. Every row is a separate balanced event.

| Event | Debit | Credit | Document/allocation result |
| --- | --- | --- | --- |
| Approve sale, net 100.00 and supplied tax 10.00 | Receivable 110.00 | Revenue 100.00; tax payable 10.00 | Invoice due 110.00 |
| Receive 60.00 allocated to that invoice | Bank 60.00 | Receivable 60.00 | Due 50.00; no second revenue/tax posting |
| Issue credit, net 20.00 and supplied tax 2.00 | Revenue 20.00; tax payable 2.00 | Receivable 22.00 | Credit balance 22.00 |
| Allocate that credit to the outstanding invoice | No new GL entry in this proposed shared-control-account model | No new GL entry | Due 28.00, credit remaining 0.00; allocation audited |
| Instead, issue the same credit after full payment | Revenue 20.00; tax payable 2.00 | Receivable 22.00 | Invoice remains paid; separate credit balance 22.00 |
| Refund that credit (a separate event) | Receivable 22.00 | Bank 22.00 | Credit remaining 0.00 |
| Approve bill, supplied cost 50.00 and tax credit 5.00 | Expense/asset 50.00; recoverable tax 5.00 | Payable 55.00 | Bill due 55.00; tax recoverability must be sourced separately |
| Pay that bill | Payable 55.00 | Bank 55.00 | Due 0.00 |

The no-new-GL allocation example is a proposed same-currency, same-control-account design, not a claim about Xero's hidden journal generation. Different control mappings, cash-basis GST timing and later FX can require additional treatment. Preserve allocations and tax attribution so Phase 2 cash/accrual reporting can be designed without rewriting original postings. Overpayment/prepayment behaviour must be decided explicitly; study does not add features to Phase 1.

### Rounding recommendation and review examples

Propose a named policy containing decimal scale, tie handling, calculation order, tax-inclusive/exclusive treatment and permitted residual handling. Snapshot the policy/version per issued document. Keep raw unit-price precision distinct from AUD settlement precision. Tax codes and account mappings are dated data; no tax constants in the posting engine.

Xero documents separate line-tax rounding and an inclusive calculation based on subtracting rounded net from rounded gross [X6]. Its example of two net lines of 45.45 with a supplied 10% test rate gives 4.55 tax each, totalling 9.10 rather than aggregate-first 9.09. This is a compatibility fixture, not a statement that every Australian sale must use that rate or method.

For an additional synthetic case, two net lines of 0.05 with a supplied 10% test rate each have unrounded tax 0.005. Proposed positive half-up line rounding gives 0.01 + 0.01 = 0.02, whereas aggregate-first gives 0.01. That one-cent difference must be visible in golden tests. A full credit reverses the saved 0.02, not a freshly computed aggregate 0.01.

ATO's indexed tax-invoice guidance [A1] confirms nearest-cent rounding with positive half-cent upwards for a single taxable sale and identifies multiple-sale rounding alternatives. Direct retrieval returned HTTP 403; complete multiple-sale wording and applicability were not verified. Therefore the proposed AU per-line default remains **UNCERTAIN pending full official-source review and Anujan's provisional decision approval**. Do not label Xero compatibility as legal validation.

We have not established Xero's complete signed midpoint behaviour from the retrieved documentation. Do not silently infer negative half-up or half-even. Recommend deriving full reversals from saved amounts and recording a separate signed-arithmetic/partial-credit rule. Supplier invoice differences should retain the supplied tax evidence and require a reviewed policy, not an unrestricted rounding account adjustment.

## First tests to specify after decision approval

| Named fixture / property | Expected invariant |
| --- | --- |
| balanced-draft-commit / posted-row-mutation-denied | Unbalanced draft rejected at commit; posted row update/delete rejected through direct SQL too |
| reversal-is-exact-inverse | Original plus reversal nets zero on every account/tax dimension even after policy/date changes |
| late-retry-no-second-posting | Same source event after replay-cache expiry yields one posting; changed payload rejected |
| period-lock-post-race | Either posting commits before the lock, or rejects after it; never a posting into an already locked period |
| invoice-receipt-credit-refund | Each numeric example above balances; receivable/credit balances reconcile; allocation does not double-post |
| credit-against-paid-invoice | Credit creation allowed within original eligible amount less prior credits; allocation cannot exceed open due |
| line-vs-total-rounding / tax-inclusive-residual | Preserve the stated one-cent differences and exact net + tax = gross for the approved calculation policy |
| partial-credit-remainder / negative-tie | Explicit approved expected outputs required; total cumulative credits never exceed original saved eligible amounts |
| tenant-isolation-allocation | Cross-tenant document/account/tax/actor references fail even with valid UUIDs |

These are proposed test cases, not tests already run. Upstream suites were inspected only; neither product was installed or executed.

## Decisions and evidence gaps for Anujan

- Approve the starting direction separately for ledger, document posting and tax policy; approval of this research PR alone must not count as approval to code provisional rules.
- Resolve the exact partial-credit tax/residual method, signed midpoint behaviour, supplier tax overrides and paid-document amendment scope in the future decisions register, with official evidence and worked examples.
- Obtain the full applicable ATO multiple-sale rounding guidance before approving the AU default. No verified conflict with Xero is asserted from incomplete retrieval.
- Xero internal database constraints and ledger storage are not public evidence here. Its full Manual Journals page could not be extracted (JavaScript shell); only the status/type and journal interfaces above are confirmed.
- Project-specific gaps and issue context are recorded in the six notes. Reported upstream issues are not reproduced defects in the pinned commits. Neither project is evidence of our PostgreSQL RLS, concurrency or production readiness.

## Public-source register for this comparison

Checked 2026-10-09 via public web retrieval/search, without account access. Where direct pages returned a JavaScript shell, usable evidence came from indexed text of the same official page; no live API was called. Links are for human rechecking and must be revalidated before implementation.

| ID | Official source | Evidence used / limitation |
| --- | --- | --- |
| X1 | [Xero Journals](https://developer.xero.com/documentation/api/accounting/journals/) | Source identifiers and signed journal amounts; no internal implementation claim |
| X2 | [Xero Types and Codes](https://developer.xero.com/documentation/api/accounting/types/) | Invoice/manual-journal statuses and journal source types |
| X3 | [Xero Invoices](https://developer.xero.com/documentation/api/accounting/invoices) | Draft/submitted versus approved posting, status changes, restricted paid edits and void example without payments |
| X4 | [Xero Credit Notes](https://developer.xero.com/documentation/api/accounting/creditnotes/) | Approved credits, outstanding-invoice allocation, remaining credit; refund via payments |
| X5 | [Xero Payments](https://developer.xero.com/documentation/api/accounting/payments) | Settlement/refund and delete-as-reversal; payments not modified in place |
| X6 | [Xero Tax and the API](https://developer.xero.com/documentation/guides/how-to-guides/tax-in-xero/) | Per-line cents and inclusive-tax calculation; no exhaustive signed-tie contract |
| X7 | [Xero tax integration guidance](https://developer.xero.com/documentation/best-practices/data-integrity/taxes/) | AU restriction on generic separate tax-total line workaround |
| A1 | [ATO tax invoices](https://www.ato.gov.au/businesses-and-organisations/gst-excise-and-indirect-taxes/gst/tax-invoices) | Official indexed excerpts only; direct page and linked PDF returned 403; full multiple-sale method confirmation outstanding |

The invoice-PDF task is now **PDF-001**; email delivery is **EMAIL-001**, depending on PDF-001. DOCS-xxx is reserved for handbook/plan changes. No code or new dependencies are authorised by these renamings.
