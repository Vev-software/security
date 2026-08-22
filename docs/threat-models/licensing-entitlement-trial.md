# Threat Model — Licensing, Entitlement & Trial

- Status: Draft for review
- Date: 2026-08-08
- Owner: `@Vev-software/security`
- Reviewers: `@Vev-software/architecture`, `@Vev-software/fabric-maintainers`
- Tracks: [`Vev-software/security#1`](https://github.com/Vev-software/security/issues/1)
- Applies to: the VEV entitlement flow defined in `engineering/handbook/09-Licensing-and-Entitlements.md`

> This is posture/governance documentation — *understand, prioritise, prove*
> (`VEV_Security_Pillar_Solution` §4). It reads the entitlement flow through its
> published contracts; it introduces **no runtime dependency** and is never on a
> product's request path (pillar §2 lens discipline).

---

## 1. Why this document exists

The commercial free/paid boundary across every VEV product **is** the entitlement
decision (`handbook/09 §1`, `15 §2`). Forging or freezing an entitlement snapshot,
rolling back a clock, or resetting a trial converts directly into unpaid use of paid
capabilities. That makes licensing/entitlement/trial the single highest-value target
in the ecosystem.

`15 §6` mandates a **threat-model note per capability**, and the Assurance pillar sells
"controls proven to be operating" (`VEV_Security_Pillar_Solution` §5). Licensing must
therefore be exemplary. This document is the consolidated attacker view and the
**normative security requirements** that the implementation issues must satisfy.

It does not re-specify the mechanism — that lives in `handbook/09`, `handbook/06` and
`VEV_Fabric_Billing_Adapter_Design`. It specifies what must be *true* for the mechanism
to be safe, and where each control lives.

## 2. System under analysis

The flow (`handbook/09 §6`, billing-adapter design §2):

```
vendor  ──▶ adapter ──▶ CommercialEvent ──▶ append-only ──▶ signed     ──▶ local
(Stripe/           (the PORT:       ledger        snapshot        evaluator
 Paddle/            vendor types    (auditable,    (JWS/Cosign,    (fast, offline,
 manual/            never cross)    replayable)    expiry+grace)   fail-static)
 marketplace)
```

Deployment modes that change the attacker's position:

| Mode | Control-plane reachable? | Clock trust | Host trust |
|------|--------------------------|-------------|------------|
| VEV-hosted (managed) | yes | VEV-controlled | VEV-controlled |
| Enterprise self-hosted (connected) | usually | customer-controlled | customer-controlled |
| Air-gapped / offline | no (imported document) | customer-controlled | customer-controlled |

The offline and self-hosted modes are the hard cases: **the host and its clock are
outside VEV's control**, so the evaluator must defend itself with only the signed
snapshot and locally persisted state.

## 3. Assets

- **A1 — Entitlement decisions.** The allow/deny + reason a product acts on.
- **A2 — Signed snapshots / offline licence documents.** The portable grant.
- **A3 — Signing keys & trust anchors.** The root of all trust in A2.
- **A4 — The append-only entitlement ledger.** The auditable source of truth.
- **A5 — Trial eligibility state.** Who has consumed a trial.
- **A6 — Time.** `expiresAt`/`graceUntil` are only meaningful against trustworthy time.

## 4. Trust boundaries

- **TB1** — vendor ⇄ adapter: untrusted webhook input crosses into VEV. Verify signature.
- **TB2** — the **port** (`CommercialEvent`): vendor types must never cross it.
- **TB3** — control plane ⇄ data plane: the runtime never does a synchronous lookup into
  the control-plane DB on the request path (`handbook/06 §1`, E6).
- **TB4** — issuer ⇄ evaluator: the snapshot signature is the only thing that carries
  trust across this boundary. Everything the evaluator receives is otherwise untrusted.
- **TB5** — evaluator ⇄ host OS/clock: in self-hosted/air-gapped modes the host is a
  potential adversary (A6).

## 5. Attacker model

- **AM1 — Paying-but-adversarial customer.** Wants to extend a trial, keep an expired or
  downgraded entitlement, or share one licence across deployments. Controls their own host
  and clock. *Primary threat actor for offline modes.*
- **AM2 — Network attacker.** Can tamper with or replay snapshots in transit (TB4).
- **AM3 — Forger.** Wants to mint a snapshot without a signing key (TB4/A3).
- **AM4 — Insider / key compromise.** Obtains a signing key (A3).
- **AM5 — Fraudulent signer-up.** Automates trial farming across identities/domains (A5).

Out of scope here: detection & response, SIEM/EDR/SOC — deliberately excluded by the
pillar (`VEV_Security_Pillar_Solution` §4). This document is prevention/assurance.

## 6. Threat catalogue (STRIDE over the flow)

Each threat maps to a required control and the issue that owns implementing it.

| # | Threat | Boundary / actor | Required control | Owned by |
|---|--------|------------------|------------------|----------|
| **T1** | **Snapshot forgery** — mint a snapshot granting paid capabilities | TB4 / AM3 | Signature verification against a published trust anchor; reject unsigned or unknown-key snapshots | `fabric#4`, private control plane, private enterprise host |
| **T2** | **Snapshot tampering** — edit capabilities/limits/expiry in transit or at rest | TB4 / AM2 | Detached signature over canonical bytes; verify **before** use | `fabric-conformance#1` |
| **T3** | **Rollback / replay** — re-present an older, more-generous or not-yet-expired snapshot after a downgrade or expiry | TB4/TB5 / AM1 | Persist highest-seen `issuedAt` + monotonic counter; refuse strictly older snapshots | **`fabric#9`** |
| **T4** | **Clock manipulation** — roll the host clock back so an expired snapshot/trial reads valid | TB5 / AM1 | Trusted-time strategy; forward-only observed-time watermark; grace bound to wall-clock advance | **`fabric#9`** |
| **T5** | **Grace-window abuse** — deliberately block the control plane to ride `graceUntil` | TB3 / AM1 | Bounded grace; grace consumed against last valid `issuedAt`, not first-outage time; telemetry on degraded mode | `fabric#4` |
| **T6** | **Trial farming** — repeated self-serve trials per identity/tenant/domain; trial resets | TB1 / AM5 | One active/consumed trial per subject; fraud signals; durable consumed-trial record | **private control plane** |
| **T7** | **Trial fail-static-open** — a trial "freezes and keeps granting" on outage like a paid plan | TB4 / AM1 | `source: trial` = **hard-stop**: zero/short grace, deny on expiry, never freeze-open | **`fabric#9`** + **private control plane** |
| **T8** | **Offline-licence sharing** — one signed enterprise document reused across many deployments | TB4 / AM1 | Bind snapshot to tenant/deployment identity; document non-transferability | private control plane, private enterprise host |
| **T9** | **Key compromise / no rotation** — signing key leaks, no revocation path | A3 / AM4 | Key rotation, trust-anchor publication, revocation / short-lived keys (Sigstore-style, `05 §6`) | private control plane |
| **T10** | **Silent grant** — an unmapped plan/price grants something instead of nothing | TB1 | Unmapped → dead-letter loudly, never grant (`09 §6`, billing design §5/§10) | private control plane |
| **T11** | **Downgrade not applied** — a cancellation/expiry event is dropped, capability lingers | TB1 | Reconciliation against the vendor API as source of truth heals drift | private control plane |
| **T12** | **Plan-check bypass** — a product uses `if (plan == …)` / `if (trialExpired)` instead of asking | TB3 | Fitness check bans plan/trial branches; decision + reason only (`09 §1`, `15 §2`) | `fabric#4`, `atlas-community#21` |

## 7. Non-negotiable security requirements (normative)

These are the invariants a product review and the conformance kit check.

1. **R1 — Verify-before-use.** The evaluator never acts on an unverified snapshot. Unknown
   key, bad signature, or malformed snapshot → **deny with reason**, never grant. (T1, T2)
2. **R2 — Fail-static is fail-static-to-last-known, not fail-open.** An outage freezes the
   last *valid* snapshot until grace expires and never silently grants a capability that was
   not in it (E6). (T5)
3. **R3 — Trials are hard-stop; paid plans are fail-static-open.** The "never stop authorised
   production" rule (`09 §4`) applies to **purchased** entitlements only. A trial must **not**
   be kept alive by an outage; on expiry it denies. (T7)
4. **R4 — Anti-rollback for offline.** Offline/air-gapped evaluation must resist clock
   rollback and snapshot replay via a persisted, forward-only watermark. (T3, T4)
5. **R5 — No silent grant, ever.** Unmapped or again-denied capabilities dead-letter; they
   never resolve to *allow*. (T10, T11)
6. **R6 — Decision + machine-readable reason only.** No product-side plan/trial booleans; the
   product renders the reason code it is given. (T12)
7. **R7 — Non-transferable grants.** A snapshot/offline document is bound to a tenant/
   deployment identity and is not portable to another deployment. (T8)
8. **R8 — Rotatable, revocable trust root.** Signing keys rotate; trust anchors are published;
   a compromised key can be revoked without re-issuing every deployment by hand. (T9)

## 8. Where each control lives (repository boundaries)

Consistent with `handbook/02 §7` dependency direction and `05 §2–§3` scope:

| Layer | Repo | Visibility | Owns |
|-------|------|-----------|------|
| Contract, taxonomy, local evaluator, snapshot format, verification, anti-rollback & trial semantics | `fabric` | public (Apache-2.0) | R1, R3, R4, R6 |
| Issuance, signing, key management, billing→entitlement, trial provisioning & anti-abuse | *private control plane* | private | R5, R7, R8, T6 |
| Offline trust-anchor config, enterprise verify host, offline licence | *private enterprise host* | private | R1, R7 (offline) |
| Conformance vectors (signature, expiry/grace, fail-static, rollback) | `fabric-conformance` | public | verifies R1–R4 |
| **Threat model, security requirements, control validation** | `security` | public | **this document** |

Names of the private repositories that own the two rows above are intentionally
omitted from this public document (`engineering/AGENTS.md` §3–§4: no private-repo
names or implementation notes about proprietary repos in public documentation).

Billing/invoicing/payments are **not** Fabric — only the entitlement model is (`05 §3`).

## 9. Control-validation checklist (for the Assurance tier)

The Assurance lens should later assert these against the audit ledger — controls *proven to
be operating*, not merely asserted (`VEV_Security_Pillar_Solution` §5). Each maps to a
requirement in §7.

- [ ] Every applied snapshot has a verified signature from a currently-trusted key (R1, R8).
- [ ] Rejected snapshots (bad/unknown signature, malformed) are denied and audited, never
      granted (R1).
- [ ] Rollback/replay attempts (older `issuedAt`/counter than watermark) are rejected and
      audited (R4).
- [ ] Clock-regression events are detected and do not extend validity (R4).
- [ ] Trial expiry denies on outage; purchased plans freeze-open within grace only (R3).
- [ ] Grace never exceeds the configured bound measured from last valid `issuedAt` (R2).
- [ ] Unmapped/again-denied capabilities appear in the dead-letter queue, never as grants
      (R5).
- [ ] No product code branches on plan name or trial-expiry boolean (fitness check) (R6).
- [ ] Offline documents are rejected on a deployment they were not issued for (R7).
- [ ] Key rotation and revocation are exercised and audited (R8).

## 10. Residual risk

- **Fully air-gapped time trust is inherently limited.** With no network heartbeat, the
  strongest available lower bound on time is the signed `issuedAt` plus a forward-only
  observed-time watermark. A determined AM1 who also wipes local evaluator state can reset
  the watermark; this is mitigated (tamper-evident state, bind to deployment identity) but
  not eliminated, and is accepted as a bounded risk for air-gapped enterprise contracts.
- **Trial anti-abuse is probabilistic.** Fraud signals raise the cost of farming; they do
  not make it impossible. The hard guarantee is one *active* trial per subject and no
  fail-static extension (R3), which bounds the value of any single farmed trial.

## 11. Related work

- Handbook: `09-Licensing-and-Entitlements`, `06-Control-Plane`, `05-The-Fabric`,
  `15-Product-Guidelines`.
- Architecture: `VEV_Fabric_Billing_Adapter_Design`, `VEV_Security_Pillar_Solution`.
- Issues: `security#1` (this model), `fabric#4` / `#7` / `#8` / `#9`,
  `fabric-conformance#1`, `atlas-community#21`. Implementation tracking for the
  private control plane and the private enterprise host is internal to those
  repositories, consistent with the public/private boundary they already observe.

---

*VEV — Engineering clarity.*
