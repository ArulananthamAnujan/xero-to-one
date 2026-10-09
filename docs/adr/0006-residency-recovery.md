# ADR-0006: Australian residency and measured regional recovery

- Status: PROPOSED — Anujan approval pending; documentation only, no infrastructure apply, account creation or spend.
- Date / owner: 2026-10-09 (Australia/Sydney) / Anujan, product and tech lead.
- Authority: approved Part B4–B5, AGENTS.md §§7, 9, BUILD_PLAN.md §§3, 7. This ADR preserves existing policy and proposes its implementation boundary.

## Decision proposed

Use Sydney (`ap-southeast-2`) as primary and Melbourne (`ap-southeast-4`) as the proposed Australian recovery region. All customer data, backups, logs and identity data must remain in Australian AWS regions. Any overseas subprocessor, including email or AI, requires Anujan's written approval in a specific ADR before use. A region label is not sufficient evidence for a provider's support access, telemetry, control-plane metadata or subprocessors.

Maintain RPO <=5 minutes and RTO <=1 hour as engineering targets, including the handbook's full-region recovery scenario. They are neither measured achievements nor customer promises. Approval does not relax these targets or authorise an unproven claim that the architecture meets them.

## Data boundary and recovery inventory

| Asset | Required design and recovery evidence |
| --- | --- |
| PostgreSQL | Sydney deployment spanning availability zones; encrypted PITR backups and cross-region backup copies to Melbourne. Retain daily backups 35 days and monthly snapshots 7 years as required by the handbook; prove restore and schema compatibility. |
| Documents and attachments | Australian regional object stores, versioned objects, encrypted Australian backup/replica copies and immutable document IDs/checksums. Confirm a restored database never silently serves a missing or different attachment. |
| Identity | ADR-0003 provider selection must prove Australian storage/processing and supported recovery for credentials, MFA/passkeys, configuration and tenant membership mappings. A user export alone is not evidence of recoverable authentication. |
| Audit, logs and traces | Australian regional destinations and backups; redact personal/financial payloads from application logs. Preserve audit history and link it to the restored database recovery point. |
| Secrets, keys and configuration | Region-local secrets and usable destination-region KMS keys, access policies, certificates and deployment artefacts. Restore tests must include key decrypt permission, not only object existence. |
| Queue | Rebuild from PostgreSQL outbox/inbox state under ADR-0004. Restore deduplication before starting workers; queue snapshots alone cannot certify completed side effects. |

Before any cloud apply, record each selected service's region, data classes, replica/export destinations, logging/support paths and subprocessors. Deny non-Australian resource destinations in reviewed deployment policy; inspect exceptions for global control-plane services individually. Never put customer data in public CI artefacts, source repositories or overseas error analytics. Synthetic local development is not permission to copy production data into agent tools.

## Recovery feasibility and known gaps

- AWS explicitly supports Sydney–Melbourne cross-region RDS automated backup replication [R1]. PostgreSQL engine/deployment support must be checked for the selected version and topology; Multi-AZ DB instances and Multi-AZ DB clusters are different products.
- RDS uploads transaction logs every five minutes and reports `LatestRestorableTime` [R2]. Cross-region transfer adds lag. Therefore backup availability alone does not establish a <=5-minute regional RPO; measure the destination recovery point, not just source backup success.
- Recommend evaluating a warm Melbourne PostgreSQL read replica to reduce promotion time and data lag alongside retained backups. Cross-region replica support is documented [R3], but the exact engine/version/instance and region combination, lag, promotion duration and cost remain unverified implementation prerequisites. Asynchronous replication does not guarantee the target.
- S3 Replication Time Control targets replication within 15 minutes, not five [R4]. Plain cross-region replication therefore cannot substantiate our whole-product RPO. The attachment design must prove a <=5-minute recoverable-object lag or use a reviewed dual-region durability acknowledgement before reporting a file as durably accepted. That choice and latency/cost need an expanded brief; no guarantee is assumed here.
- Identity recovery remains coupled to ADR-0003. An available database with unavailable login is not a recovered product. MFA/passkey recovery, revocation, issuer/endpoints and failover sessions must be proved before claiming RTO compliance.
- A cold snapshot restore can exceed one hour depending on size and dependencies. Warm standby reduces some work but costs more; budget approval, service quotas and staffing remain unresolved. Retained backups remain necessary for corruption/ransomware even with a live replica.

## Proposed human-operated runbook

1. Record outage start, affected services and last known durable recovery points; page the authorised operator. Include detection and decision time in total recovery measurement.
2. Fence the old writer and pause side-effect workers before promotion. Prevent two active write regions; inability to establish fencing is a safety stop requiring operator judgment.
3. Select the recovery point, promote a verified replica or restore to a new database, and reapply the reviewed access/network settings. Check destination keys, object versions and identity readiness.
4. Verify trial balance, journal/source-event uniqueness, tenant isolation, audit continuity and attachment checksums. Reconcile any provider outcomes beyond the restored database point before replaying email/payment-related work.
5. Switch approved routing, validate login/read/write paths with synthetic records, then resume idempotent workers from durable completion state. Record actual lost interval and time to restored service.
6. Keep the former writer fenced; failback is a separately reviewed operation. Do not merge divergent financial histories automatically.

## Validation and acceptance evidence

- Run the handbook's monthly real restore using synthetic representative volumes; record database size, event/attachment counts, recovery point, missing objects, elapsed stages and total elapsed time.
- `region-loss-recovery`: Sydney unavailable; restore usable authenticated ledger/attachments in Melbourne; observed loss <=5 minutes and total recovery <=60 minutes, or report failure explicitly.
- `corruption-pitr`: recover before synthetic corruption without propagating it from a live replica; verify ledger/audit and object references.
- `queue-replay-after-restore`: duplicate deliveries cannot repeat an internal financial effect; ambiguous external outcomes remain held for reconciliation.
- `residency-inventory`: every data-bearing store, export, replica, identity path and telemetry destination has verified Australian residency or an explicit written subprocessor exception; no exception is granted here.
- `key-and-identity-loss`: validate decrypt permissions, authorised operator access and end-user MFA/login in the recovery region, with no disabled security controls.

No infrastructure exists or drill was run for this ADR. Targets remain unproven until evidence is reviewed; failure requires remediation and reporting to Anujan, not quietly changing the target. No real pilot bookkeeping before the separate accountant sign-off gate.

## Alternatives, consequences and approval

Single-region Multi-AZ alone does not address a complete region loss. Backup-only regional recovery is simpler and cheaper but cannot presently substantiate the required RPO/RTO. Proposed warm recovery adds cost and operational work; exact capacity and budget are an Anujan decision before provisioning. Global or overseas replicas are excluded by current residency policy.

Approval requested: Sydney primary, proposed Melbourne recovery boundary and the measurement/gating approach. Before infrastructure approval, resolve the warm-standby budget, pinned service availability, attachment durability and identity recovery design. No new software dependency is introduced by this docs-only proposal.

## Primary sources checked 2026-10-09

- R1: [RDS cross-region backup support and region-pair matrix](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReplicateBackups.html) and [engine/version support](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RDS_Fea_Regions_DB-eng.Feature.CrossRegionAutomatedBackups.html).
- R2: [RDS point-in-time restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html) — log upload cadence and latest restorable time; not a whole-system guarantee.
- R3: [RDS cross-region read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.XRgn.html) — prerequisites and regional operation.
- R4: [S3 Replication Time Control](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-time-control.html) — replication time objective differs from this platform's target.
