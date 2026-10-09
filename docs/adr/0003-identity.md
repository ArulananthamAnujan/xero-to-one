# ADR-0003: Phase 1 identity provider

- Status: **PROPOSED — Anujan decides; no provider selected or deployment authorised**.
- Date / evidence checked: 2026-10-09 (Australia/Sydney).
- Owners: Anujan, product lead and tech lead; security reviewer independently checks the implementation brief.
- Scope: compare Cognito in `ap-southeast-2` and self-hosted Keycloak in Australian AWS regions. No software installed, accounts created or provider contacted.

## Context and recommendation

AGENTS.md §§2, 3 and 7 require MFA, passkeys, sensitive-action re-authentication, tenant isolation, Australian identity/log/backup residency and a small-team operating model. ADR-0002 owns tenant-local business users; provider identity never grants tenant access by itself. ADR-0006 owns recovery targets.

**Recommend Cognito Plus in Sydney, conditionally**, because managed patching/scaling reduces Anujan's operating load and Plus includes compromised-credential detection, adaptive protection and authentication-event export [S1, S2]. This recommendation is blocked from implementation until the residency, passkey assurance and identity-recovery gates below are resolved. If Cognito cannot meet those gates, return this ADR to Anujan for a Keycloak decision; do not silently weaken the requirements or deploy either option.

## Comparison

| Concern | Cognito Sydney | Self-hosted Keycloak in Australian AWS regions |
| --- | --- | --- |
| MFA and passkeys | Password plus TOTP; Essentials/Plus support WebAuthn. Current docs permit verified passkeys to satisfy required MFA with `FactorConfiguration=MULTI_FACTOR_WITH_USER_VERIFICATION`. Passkeys are not a password second factor. Email/SMS passwordless OTP cannot coexist with required MFA [S3]. | Configurable OTP and WebAuthn/passwordless flows, user-verification policy and passkey enrolment. We own correct mandatory-flow configuration, recovery and upgrades [S4]. |
| Step-up | Essentials/Plus support `acr_values`/`max_age` in managed login and `TARGET_ACR_VALUES`/`MAX_AGE` in `USER_AUTH`. Backend validates achieved `acr`/`amr` and `auth_time`, not requested assurance; refresh does not reset authentication freshness [S5]. Regional flow configuration still needs validation. | OIDC ACR/level-of-authentication flows and max-age controls support step-up. API must verify achieved assurance and freshness, not trust requested `acr_values` [S4]. |
| Australian residency | Profile data stays in pool region, but optional email/analytics routing can cross regions. Custom-domain login uses CloudFront and a US-region certificate: pool placement alone is insufficient evidence [S6, S7]. | Choose Sydney compute/database/keys/logs and Melbourne recovery storage; we control configuration and egress. AWS global services, mail routing, client authenticators and support paths still need review. Self-hosting is not an automatic residency guarantee. |
| Operations | AWS operates the identity service; we still own policy, IAM, quotas, key/client rotation, incident handling, logs, recovery tests and application sessions. | We patch Keycloak/JVM/container and database, rehearse upgrades/rollback, monitor clusters, back up secrets and credentials, rotate keys and provide on-call coverage. HA includes database, network and operator procedures [S8]. |
| Scale/cost | MAU charge with optional add-ons; no server fleet. Plus costs below are identity-service fees, not total hosting. | No MAU licence charge; compute/database/HA and operator time scale with authentication traffic, not registered-user count alone. Load tests determine capacity [S9]. |
| Lock-in | OIDC helps application portability, but AWS-specific policies, APIs, claims, recovery and credential migration remain work. Do not assume password hashes or enrolled credentials can be exported. | Apache-2.0 project [S10]; standards and owned database increase control, but realm flows, extensions, upgrades and credential formats create migration work. Avoid custom authenticators unless separately approved. |

### Mandatory MFA compatibility decision

Do not rely on older statements that Cognito passkeys always exclude MFA: current API/documentation describes the verified-passkey exception [S3]. Proposed policy is mandatory password+TOTP **or** user-verified passkey, with no email-OTP/password-only fallback. Confirm feature availability in Sydney, supported clients and assurance returned to the backend before implementation approval. Test missing user verification, bypassed enrolment, recovery, remembered-device behaviour and stale sessions. Treat synced-passkey provider storage as a separate residency question; prefer device-bound credentials until the boundary is approved. No new SMS or email provider is authorised here.

### Application boundary

Use a replaceable identity adapter and backend session layer; validate issuer, audience/client, signature, expiry and expected token purpose. Resolve provider subject to an authorised tenant-local user through the ADR-0002 selection mechanism. Never use email equality, provider groups or a submitted organisation ID as sufficient membership evidence.

Preserve the handbook's 15-minute access lifetime, rotating refresh tokens, HttpOnly/Secure/SameSite cookies, 30-minute idle timeout and session revocation. Sensitive actions require a fresh, short-lived application proof bound to user, active organisation, session and operation. Exact proof lifetime and provider assurance mapping belong in an approved API-002 brief. Cognito levels are provider-specific: inspect permitted methods as well as the level; do not assume a numeric level alone proves our verified-passkey policy. Reject missing/insufficient claims or stale `auth_time`, including after refresh [S5]. Recovery and factor replacement must not bypass these requirements.

## Cost comparison: explicit planning scenarios

USD/month, excluding tax and currency conversion; checked 2026-10-09. Assume 1,000 or 100,000 registered users **all active that month**, direct user-pool login, one production directory, no federation, no M2M and no purchased quota increase. Essentials assumes its 10,000 free MAUs are available across the AWS account/organisation. A user across client organisations is not automatically multiple MAUs; identity mapping and pool layout affect counting.

| Option | 1,000 MAU | 100,000 MAU | Basis and exclusions |
| --- | ---: | ---: | --- |
| Cognito Essentials | $0 | $1,350 | `max(MAU−10,000,0) × $0.015`; lacks Plus threat features [S1]. |
| Cognito Plus (recommended) | $20 | $2,000 | `MAU × $0.020`; no free tier; threat capabilities included rather than the legacy Lite ASF surcharge [S1]. |
| Keycloak software | $0 | $0 | Apache-2.0 community software; no purchased support assumed [S10]. |
| Keycloak example compute floor | $86.43 | $259.30 | 730 hours: respectively 2 tasks × (1 vCPU, 2 GB), or 3 × (2 vCPU, 4 GB); Sydney Linux/x86 Fargate rates $0.04856/vCPU-hour and $0.00532/GB-hour [S11]. Proposed capacity, not benchmarked. |
| Keycloak illustrative complete budget | $1,586.43 | $4,659.30 | Compute above + assumed $300/$800 other infrastructure + 8/24 operating hours at assumed $150/hour. These allowances are planning assumptions, **not AWS quotes**. |

The Keycloak allowance must cover HA PostgreSQL, load balancer, storage/backups, Australian recovery copies, network, logs/metrics and keys; it may be insufficient. Replace it with a regional calculator estimate and load test before spend approval. Traffic assumption for that future test: 20 full sign-ins and 200 refreshes per active user/month, then test concentrated peaks and failover. MAU alone cannot validate those fleet sizes [S9]. Cognito also needs operator time, mail, logs, keys, network and recovery costs; its fee rows are not directly comparable with the illustrative Keycloak total. For initial budgeting, assume 4/8 Cognito operating hours at $150/hour, yielding $620/$3,200 **plus unpriced ancillary services** for Plus; this is an explicit estimate, not a measured saving.

Replication is separately priced and is **not included**. The public pricing page gives an Essentials replication example at $0.0045/MAU; do not apply that rate to Plus without confirming its quote. No old introductory credits or grandfathered Lite pricing are assumed [S1].

## Residency and recovery gates before implementation

1. **Anujan/security:** approve a complete identity data-flow inventory: attributes, credential material, tokens, IP/device metadata, audit events, backups, mail, support access, telemetry and passkey sync. Document any global metadata/control-plane handling; no overseas subprocessor or exception is approved by this ADR.
2. **Planner/security:** establish an Australian login path. Custom-domain CloudFront/US ACM is not pre-approved; examine regional API-based flows as an alternative, including their different step-up implementation. Do not write custom cryptography. Disable unapproved analytics, social federation and overseas log destinations [S6, S7].
3. **Planner/Anujan:** resolve Cognito recovery. Current replication docs require eligible modern pools and state that secondary replicas do not support TOTP MFA, sign-up or password reset [S12]. Thus replication does not demonstrate a one-hour recovery for all permitted users. Verify an Australian secondary region and restore/failover behaviour before claiming ADR-0006 targets; profile exports are not assumed to back up usable credentials.
4. **Test/security:** after separate infrastructure authorisation, prove password+TOTP and user-verified-passkey flows, fresh step-up, session revocation, lost-factor recovery and tenant switching. Negative cases must deny protected commands. No live cloud test was performed for this ADR.
5. **Anujan:** confirm the conditional provider/tier choice and cost assumptions. If unresolved requirements require policy changes or a different provider, amend this ADR explicitly before implementation.

## Official evidence

All sources below checked 2026-10-09; dynamic docs/prices must be rechecked for the implementation brief. No reference-project source code was read or reused in this ADR.

- [S1 — Cognito pricing](https://aws.amazon.com/cognito/pricing/): tiers, MAU examples, free-tier scope, add-ons and separately billed email/SMS.
- [S2 — Cognito feature plans](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-sign-in-feature-plans.html): feature/tier eligibility.
- [S3 — MFA rules](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa.html) and [WebAuthn configuration](https://docs.aws.amazon.com/cli/v1/reference/cognito-idp/set-user-pool-mfa-config.html): verified-passkey MFA exception.
- [S4 — Keycloak administration](https://www.keycloak.org/docs/latest/server_admin/index.html): WebAuthn, passkeys and step-up authentication.
- [S5 — Cognito authentication levels and step-up](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-step-up-authentication.html): managed/API parameters, achieved claims, tier requirements and refresh freshness. `prompt=login` alone requests re-authentication, not a specific assurance level.
- [S6 — Cognito regional data considerations](https://docs.aws.amazon.com/cognito/latest/developerguide/security-cognito-regional-data-considerations.html): profiles and optional outbound routing.
- [S7 — Cognito custom domains](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-add-custom-domain.html): CloudFront and certificate requirements.
- [S8 — Keycloak HA responsibilities](https://www.keycloak.org/high-availability/introduction).
- [S9 — Keycloak sizing](https://www.keycloak.org/high-availability/multi-cluster/concepts-memory-and-cpu-sizing): traffic-based estimates, database load and headroom; not a benchmark of our proposed deployment.
- [S10 — Keycloak project licence declaration](https://github.com/keycloak/keycloak): Apache-2.0; pin/check chosen release and dependency evidence before installing.
- [S11 — AWS Sydney ECS/Fargate public price list](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonECS/current/ap-southeast-2/index.json): on-demand Linux/x86 vCPU/memory dimensions, retrieved read-only.
- [S12 — Cognito multi-Region replication](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-multi-region.html): eligibility, eventual consistency and secondary limitations.
