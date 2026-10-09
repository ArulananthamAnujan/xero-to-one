# ADR-0004: Valkey-backed background jobs with a durable outbox

- Status: PROPOSED — Anujan approval pending; no dependency or infrastructure installed.
- Date / owner: 2026-10-09 (Australia/Sydney) / Anujan, product and tech lead.
- Authority: AGENTS.md §§3, 6–9; BUILD_PLAN.md §§3, 5; approved Part E. This proposal does not change phase scope, tenant isolation or accounting approval gates.

## Context and recommendation

Recommend BullMQ Community with Valkey for background dispatch/execution, with PostgreSQL as the durable authority for pending work and completion. Phase 1 examples are PDF generation, approved email delivery and attachment processing; mentioning a queue does not authorise Phase 2 bank/ATO jobs early.

Valkey is BSD-3-Clause and BullMQ Community is MIT [Q1–Q2]. Current BullMQ connection documentation describes Valkey GLIDE support, and the maintainer benchmarks BullMQ with several Valkey versions [Q3–Q4]. These support evaluating the pairing, not a guarantee for an unpinned client/server/managed-service combination. Select exact supported versions and client in the dependency-evidence PR; no adapter is added by this ADR.

Prefer a managed AWS Valkey deployment in Sydney for the future production operating model, with a separate queue instance from evictable application caching. Hosting SKU, engine version, topology, Melbourne recovery availability and costs remain implementation prerequisites, not an infrastructure purchase approval. Follow the chosen service's supported durability settings; do not assume self-hosted AOF controls exist in a managed service.

## Delivery contract

1. Commit business action, balanced ledger posting where applicable, audit and outbox row in one PostgreSQL transaction. Never contact a provider inside that transaction.
2. A dispatcher works one authenticated service-authorised organisation context at a time under RLS, leases eligible rows and enqueues a stable event identifier. Its tenant schedule must not query cross-tenant business rows or bypass RLS.
3. A successful enqueue is not final delivery. Retain durable pending/completion state; a reconciler reschedules events lacking durable completion after lease expiry, including after total queue loss.
4. Queue messages contain tenant/event identifiers and schema version, not invoices, identity secrets or customer attachments. A worker validates the envelope against the persisted event and establishes an authorised tenant context; the message's tenant ID alone grants no authority.
5. Use a database consumer-inbox uniqueness key scoped to organisation, event and handler. Commit inbox completion with internal domain effects. Retain permanent financial source-event uniqueness independently of BullMQ job retention and HTTP response-cache expiry.
6. Expect at-least-once delivery. A worker can crash after a provider accepts a request but before recording success: pass a stable provider idempotency key where supported; otherwise mark an ambiguous outcome for reconciliation, rather than blindly replaying a potentially duplicate external effect. Do not claim exactly-once email delivery.
7. Retry transient failures with bounded exponential backoff and jitter; put exhausted/permanent failures in durable failed status with redacted error classification, attempt count and next action. Authorised manual retry is audited and reuses the business event ID. Limits/timeouts are job-specific configuration in approved briefs; no new universal retry duration overrides the handbook.

## Reliability and security requirements

- BullMQ requires no arbitrary queue-key eviction; use `noeviction`, capacity alerts and admission/backpressure before memory exhaustion [Q5]. Never put this queue in the same eviction domain as disposable caches.
- Valkey replication is asynchronous and can lose acknowledged writes during failure; persistence improves restart recovery but does not replace the PostgreSQL outbox [Q6]. Reconstruct eligible work from durable state after loss; do not reconstruct financial truth from queue history.
- TLS, authentication, least-privilege ACLs/network paths and encryption at rest; separate environments and credentials. Customer-derived queue data, snapshots, logs and telemetry stay in Australian AWS regions under ADR-0006.
- Monitor oldest incomplete outbox age, lease expiry, retry/failed totals, worker stalls, queue memory and restore replay backlog. Alert on durable work age, not queue length alone.
- Graceful shutdown stops new work and releases/recoverably expires leases. CPU-heavy PDF jobs must not starve lock renewal; worker concurrency and execution timeouts need measured limits.
- A queue outage may delay side effects after an otherwise successful business commit. The UI shows durable pending/failed delivery state; do not report a successful email merely because invoice issue succeeded.

## Alternatives and consequences

| Option | Assessment |
| --- | --- |
| BullMQ + Valkey | Recommended: matches the approved queue direction and JS worker stack; adds a datastore and failure/replay operations. |
| PostgreSQL-only dispatcher | Fewer services but more scheduling/locking ownership; current BullMQ also documents a PostgreSQL backend [Q7]. Revisit only through an amended approved ADR after workload evidence, not an unsolicited stack change. |
| AWS queue service | Potentially less engine operation, but changes planned dependency/service contracts; no selection made here. |
| Self-hosted Valkey | Preserves engine control, but small-team patching, backups and failover burden makes it a less attractive production default. |

## Verification before implementation/production

- `outbox-commit-rollback`: aborted invoice transaction produces no enqueueable event; committed event survives dispatcher restart.
- `enqueue-ack-lost` and `worker-crash-after-commit`: duplicate deliveries result in one internal effect and one durable completion.
- `valkey-empty-recovery`: erase a synthetic queue, reconstruct incomplete events, preserve completed financial event deduplication.
- `tenant-envelope-mismatch`: event from organisation A with B's envelope is rejected without reading/writing B's records.
- `provider-ambiguous-result`: timeout after external acceptance enters reconciliation; no unsafe automatic duplicate send.
- `pinned-stack-conformance`: delayed jobs, retry/backoff, stalled recovery, scripts, TLS/ACLs, noeviction and failover tested against the exact BullMQ/client/Valkey/service configuration. No tests have been executed in this docs-only ADR.

Approval requested: accept Valkey/BullMQ plus durable-outbox architecture. Separate brief approval and golden-rule-9 evidence (reason, licence, weekly downloads, latest release date; explicitly N/A where meaningful) remain required before installation. Hosting cost, versions and restore behaviour are still open implementation evidence, not hidden approval.

## Primary sources checked 2026-10-09

- Q1: [Valkey licence](https://github.com/valkey-io/valkey/blob/unstable/COPYING) — BSD-3-Clause; recheck the selected release and bundled notices before installation.
- Q2: [BullMQ Community licence](https://github.com/taskforcesh/bullmq/blob/master/LICENSE) — MIT; does not authorise proprietary Pro features.
- Q3: [BullMQ connections](https://docs.bullmq.io/guide/connections) — client adapters and connection requirements.
- Q4: [Maintainer Valkey comparison](https://bullmq.io/articles/benchmarks/valkey-performance-across-versions/) — tested version examples, not our compatibility certification.
- Q5: [Production guidance](https://docs.bullmq.io/guide/going-to-production) — noeviction and connection behaviour.
- Q6: [Valkey replication](https://valkey.io/topics/replication/) and [persistence](https://valkey.io/topics/persistence/) — replication/durability limitations.
- Q7: [BullMQ PostgreSQL backend announcement](https://bullmq.io/news/260927/bullmq-v6-postgresql/) — alternative only; not selected or installed.
