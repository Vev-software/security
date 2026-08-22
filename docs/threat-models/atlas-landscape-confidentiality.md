# Threat Model — Confidentiality of the Atlas Landscape Map

- Status: Draft for review
- Owner: `@Vev-software/security`
- Reviewers: `@Vev-software/architecture`
- Tracks: [`Vev-software/security#4`](https://github.com/Vev-software/security/issues/4)
- Applies to: both Atlas editions — self-hosted **Community** and hosted **Enterprise**

> This is posture/governance documentation — *understand, prioritise, prove*
> (`VEV_Security_Pillar_Solution` §4). It reads both editions through their published
> contracts and public repositories; it introduces **no runtime dependency** and is
> never on a product's request path (pillar §2 lens discipline). Reporting a
> vulnerability follows the private disclosure path in
> [`SECURITY.md`](../../SECURITY.md) — never a public issue.

The umbrella threat model for the Atlas product line. It ties the edition-specific
hardening work into one coherent public security posture instead of two unrelated
ones. This document describes VEV's security **model and architecture**; it is not
an implementation changelog or a point-in-time status tracker, and it carries no
exploit steps or deployment topology.

## Why this document exists

The Atlas landscape map's **confidentiality is the primary security property of the
product** — every other feature is built on top of the assumption that the map does
not leak. Two editions share that same property but run on different infrastructure
(single-tenant self-hosted vs. shared hosted infrastructure), so they need related
but not identical controls. This document is the consolidated public statement of
what both editions guarantee and why, so a prospective user or customer can evaluate
the security posture without needing private, edition-specific detail.

## What Atlas holds

Atlas builds a tenant's **landscape map**: the systems, applications, servers,
infrastructure and data layer in an organisation's estate, and the relationships
between them. The map does not contain credentials or customer records, but it is
**reconnaissance-grade data** — it tells an attacker how the estate is built and
where to look. The security goal is the **confidentiality and integrity of that
map**, and a **trustworthy, tamper-evident audit trail** of who read, changed or
exported it. Every control in this document exists in service of that one goal.

## Security model

Isolation is **defense in depth**, not a single control:

1. **Identity is established, never asserted.** A request's tenant and principal
   are resolved from a verified identity layer, never taken from caller-supplied
   values. A deployment that has no verified identity provider configured **fails
   closed** — it refuses to serve requests rather than falling back to a trusting
   default.
2. **Application-layer tenant scoping holds by default.** Every read and write of
   tenant-scoped data is scoped to the caller's tenant by construction, not by a
   predicate an individual query has to remember. Isolation failing open — a query
   that silently reads across tenants — is treated as a build-breaking defect, not
   a runtime bug to catch later.
3. **The hosted, shared-infrastructure edition adds a database-enforced backstop
   beneath the application layer**, so that even a defect in the application's own
   scoping cannot produce a cross-tenant read. Isolation on shared infrastructure
   is not allowed to depend on the application layer alone.
4. **Every write and every export is authorized and audited.** Bulk access to the
   map (a full-landscape export) is a distinct, elevated, rate-limited and audited
   operation — not a side effect of ordinary read access.
5. **The AI path is governed, not incidental.** Where an AI capability can see
   landscape data, that data path is scoped to the same tenant boundary as
   everything else, egress is explicit rather than silent, and the model is never
   the sole mechanism for a security or access decision.

## Trust boundaries

- **Caller → API.** Requests are not trusted to name their own tenant, principal
  or roles; identity is established by the configured identity layer, and every
  write and every export is authorized against it.
- **API → platform substrate.** Identity, authorization, audit and entitlement
  decisions are consumed through VEV's shared platform contracts (Fabric), not
  reimplemented per product. The seam is a versioned public contract in every
  case, whether the implementation behind it runs locally (Community) or as part
  of the hosted service (Enterprise).
- **Runtime → storage.** The map is persisted data at rest, whether that is a
  single self-hosted database file or shared hosted infrastructure. In both
  cases, storage is treated as a reconnaissance-grade artifact that needs
  protecting independently of the application layer above it.
- **Image/service → operator.** Whoever runs Atlas — a self-hoster running a
  container they did not build, or VEV operating the hosted service — is
  expected to be able to verify what they are trusting before data reaches it
  (signed, provenance-carrying releases for the self-hosted image; a governed
  operations posture for the hosted service).

## Controls, by edition

### Community (self-hosted)

Fully implemented and enforced by tests that fail the build on regression. Detail
lives in the edition's own public threat-model note, linked below — this table is
a summary, not a duplicate.

| Threat | Control |
|---|---|
| A caller asserts a tenant or role it does not hold | Identity is resolved by the configured identity layer; any non-development deployment without a real identity provider fails closed rather than trusting request headers |
| One tenant reads another tenant's map | An application-layer global query filter scopes every query to the ambient tenant by default; a fitness test fails the build if the filter is bypassed without an audited, explicit exception |
| Silent bulk exfiltration of the whole map | Full-landscape export requires elevated authorization, is rate-limited per tenant, and produces exactly one audit record per export |
| No record of who changed or read what | An append-only audit envelope on every write and every export, with no secrets or customer content in it |
| A paid capability is reached in the free edition | The entitlement seam denies reserved paid capabilities in the free edition by default (fail-static); the free/paid line is data, not a code branch |
| The map is read from a stolen disk or volume | A documented encryption-at-rest expectation for the self-hosted database, alongside the same care for exports and backups |
| Running a tampered or unknown image | Tagged releases publish a signed image with a software bill of materials and build provenance; an unsigned or tampered image fails verification |
| AI chat or AI-assisted review leaks cross-tenant data | AI grounding is built through the same tenant-scoped data access as every other read path; the tenant supplies their own AI provider key, and data egress to a provider is explicit and opt-in |

Full detail, trust boundaries and the compatibility statement: the Community
edition's own public threat-model note (linked from that repository's
`SECURITY.md`).

### Enterprise (hosted)

The hosted edition runs on shared infrastructure, which is a materially different
threat surface from a self-hosted, single-tenant deployment: isolation has to hold
even when tenants share a database. Its security model adds, on top of everything
in the Community column:

| Threat | Control |
|---|---|
| A defect in the application's own tenant scoping is missed on shared infrastructure | A database-enforced, per-tenant access policy acts as a backstop beneath the application-layer filter, so cross-tenant access is not possible from that layer alone, even with the application filter deliberately disabled |
| The map is read from a compromised storage medium or backup | Data at rest is encrypted with per-tenant key material, not a single shared key |
| Discovery ingestion collects more than the map needs | Ingestion is scoped to data minimization: only what the landscape map requires is retained |
| An AI capability trains on, or leaks, a customer's landscape data | The hosted AI path does not train on customer data, keeps data resident in the committed region, redacts sensitive content before it reaches a model, and is designed against prompt-injection risk |
| An operator or support role has more access than their task needs | Fine-grained, least-privilege role-based access control governs the hosted operations/read-only portal |
| Data subject rights and regulatory obligations on hosted data | Encrypted backups, a right-to-erasure mechanism, a maintained subprocessor list and a data processing agreement cover the hosted service's GDPR obligations |

## Residual risk

- **The operator owns the perimeter around what they run.** Neither edition
  provides its own network isolation, WAF or TLS termination; the self-hoster (or
  VEV, for the hosted service) runs it behind their own ingress and access
  control.
- **A self-hosted, single-tenant deployment is not a multi-tenant boundary.** The
  Community edition serves one tenant per running instance; there is no
  cross-tenant question to answer within a single self-hosted deployment. The
  database-enforced backstop above is specifically an *additional* control for
  shared, multi-tenant infrastructure, not a claim that self-hosted deployments
  need it too.
- **Secrets and customer content are kept out of logs and audit trails by
  design**, but an export or a backup is a full plaintext copy of the map in
  either edition and must be protected in kind by whoever holds it.
- **AI features are opt-in and tenant-controlled.** No landscape data reaches an
  AI provider unless the tenant has enabled the capability; the model is never
  the sole authority for a security or access decision in either edition.
- **This document describes the intended security model, not a point-in-time
  implementation status.** Treat it as the standard both editions are held to,
  not as a disclosure of what is or is not yet live for either one.

## Related work

- Sibling threat model: [`licensing-entitlement-trial.md`](./licensing-entitlement-trial.md)
  (`security#1`) — fail-static and anti-tamper posture for the commercial gating
  layer that governs which capabilities either edition is entitled to in the
  first place. That document, not this one, is the place for entitlement-forgery
  or trial-abuse threats.
- Architecture: `VEV_Security_Pillar_Solution`.
- Issues: `security#4` (this model). Edition-specific implementation issues stay
  in each product's own repository, consistent with the public/private boundary
  those repositories already observe.

---

*VEV — Engineering clarity.*
