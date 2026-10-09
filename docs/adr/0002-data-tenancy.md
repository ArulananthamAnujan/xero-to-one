# ADR-0002 — tenant-local data and future accountant access

Date: 2026-10-09. **Status: PROPOSED — Anujan approval required.** Docs only; schema, migrations and implementation require later approved briefs. Owner/approver: Anujan.

## Decision proposed

Use one PostgreSQL database/shared schema initially, with tenant-local users and memberships. Every tenant-owned row has organisation_id; every tenant-owned reference uses a composite organisation/id foreign key. Shared catalogue/control metadata must be explicitly classified and must not contain client books or be a shortcut around RLS. No cross-tenant accounting query is permitted.

| Concern | Proposed boundary |
| --- | --- |
| Organisation | Tenant aggregate and lifecycle; business data never inferred solely from an object UUID |
| Users | Tenant-local row containing identity-provider issuer/subject binding, never email as authentication key; the same human can have separate local rows in several tenants |
| Membership/roles | Active membership binds identity to local user/permissions; validate membership and revocation state on every request/job before business access |
| Accounts, tax codes, periods, journals | Tenant-owned; composite references including actor, account, period, tax, reversal and source mappings |
| Money | Posted amounts integer minor units; unit prices NUMERIC(19,4) where needed; FX NUMERIC(24,10); AUD-only gateway in Phase 1 |
| Idempotency | Request key scoped to tenant/principal/operation and payload fingerprint; permanent tenant/source-event uniqueness distinct from 24-hour response replay |
| Audit/outbox | Tenant-scoped append path and transactional event publication; no external API call inside the posting transaction |

Keep UUIDv7 identifiers, measured tenant-leading indexes and expand/migrate/contract migrations per handbook. This ADR does not approve DDL, an ORM or a PostgreSQL major version. Knex/pg remain study recommendations pending implementation evidence and approved brief; no package installation occurs.

## Request and worker flow

1. Validate the provider-issued session cryptographically (issuer, audience, expiry and intended use). The identity ADR decides the provider; do not trust an arbitrary organisation claim supplied by a browser.
2. Tenant selection is a server-authorised session transition. The requested organisation is a selection hint only: verify its active membership and bind the resulting context to the authenticated session. Ordinary business requests never establish tenant authority from body/query parameters.
3. Begin a database transaction and set organisation context transaction-locally from that verified session. Apply explicit application filters plus RLS for both reads and writes. Missing/malformed context must deny or fail before any business row is returned.
4. Use a non-owner runtime role without BYPASSRLS/superuser/DDL/TRUNCATE privileges; force RLS on tenant tables. Migration/admin roles are separate and never used for requests. RLS is defence in depth, not protection against a fully compromised privileged application or SQL injection able to change context.
5. Commit or roll back; context must not leak through the connection pool. Workers validate a persisted tenant/job binding and execute one tenant at a time, never take an arbitrary tenant identifier from untrusted event text.

PostgreSQL documents owner/BYPASSRLS exceptions and operations not governed by RLS [P1], transaction-local configuration [P2], and multi-column constraints [P3]. Therefore RLS alone is insufficient; the role, query boundary and composite-key rules are inseparable.

## Phase 4 accountant hub — design only

One accountant authenticates once with the chosen identity provider. That issuer/subject can bind to a distinct local user and membership in each invited client organisation. It must never collapse client financial rows into a global user-owned dataset.

A future identity control-plane grant directory may return only organisations this principal is invited to access (opaque organisation handle and display label), not their business data. This directory is explicitly not a tenant-books table. Its minimal schema, residency, revocation and enumeration controls need separate Phase 4 approval; no directory implementation or cross-tenant RLS bypass is authorised here. Phase 1 may use a server-validated organisation invitation/selection path without a global multi-client dashboard.

The hub selects one grant, verifies live membership and receives a tenant-bound session/context. A second browser tab or background job has its own bound context; changing one tab must not silently retarget another command. Aggregate hub cards are composed from separately authorised, bounded single-tenant responses, never one SQL join across clients. Each result retains its tenant label; revoked grants fail closed. Bulk BAS is later fan-out to separately authorised tenant jobs with per-client audit/results, not one multi-client posting transaction.

## Ledger and transaction consequences

Ledger balance/immutability is enforced independently of tenancy. Follow deterministic locks: periods sorted, headers sorted, account/tax rows in fixed table/ID order, then lines. Re-read after locks and retry from the start on changed scope; never take an earlier lock class late. Database tests must cover direct invalid writes, not only repository methods. Full reversals copy saved amounts under the approved accounting direction; configuration never disables structural invariants.

## Alternatives and trade-offs

- Database/schema per tenant: clearer physical separation but more migrations, connections and recovery work for a small team; defer dedicated routing until measured need. This design must keep organisation routing central for that future migration.
- One shared global user row owning financial relationships: simpler profile updates, but conflicts with the approved tenant-local assumption and risks accidental joins. Reject for Phase 1; provider identity linkage does not transfer business ownership.
- Shared schema with application filters alone: rejected because one missing predicate can expose data. RLS plus composite FKs and a non-bypass runtime is mandatory.

## Future verification and unanswered details

- `cross-tenant-reference-rejected`: a valid account UUID from B cannot be attached to A's journal.
- `missing-context-denied` and `pool-context-cleared`: a reused connection with no fresh context returns no tenant data.
- `identity-email-change`: changed email does not create or transfer tenant authority; issuer/subject governs identity.
- `membership-revoked-in-flight`: define transaction/revocation linearisation in the API brief; later requests/jobs must reject revoked membership, and queued jobs cannot inherit stale human authority indefinitely.
- `two-tabs-two-clients`: accountant A/B tabs cannot cross-post; object IDs and idempotency keys remain tenant-bound.
- `accountant-hub-no-cross-tenant-query`: inspect query boundaries and per-client audit attribution; no admin role used by the hub.

These are test requirements, not implemented tests. Retention/anonymisation of identity versus immutable audit actors, bootstrap/invitation anti-enumeration and system-worker permissions need detailed approved briefs before their code. No auth provider or global directory is selected by this ADR.

## Sources (checked 2026-10-09) and approval

- P1: [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).
- P2: [PostgreSQL configuration functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-SET).
- P3: [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).
- Project authority: [AGENTS.md](../../AGENTS.md) §§3–7; [REF-001](../reference-notes/REF-001-comparison.md). Reference-project notes inform separation of document/posting concerns, not PostgreSQL tenant guarantees.

**Anujan:** approve the tenant-local model and constrained future accountant-hub direction. This grants no cross-tenant query exception and no infrastructure permission.
