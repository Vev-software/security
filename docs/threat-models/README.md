# Threat models

Public threat models for cross-cutting VEV security surfaces. Each is posture/governance
documentation (*understand, prioritise, prove*) that reads the system through its published
contracts — it introduces no runtime dependency and sits on no request path
(`VEV_Security_Pillar_Solution` §2).

Scope rule (per repo `README` and `docs/adr/0001`): these documents describe threats,
required controls and where controls live. They contain **no** secrets, exploit material,
or private incident detail.

| Threat model | Surface | Tracks |
|--------------|---------|--------|
| [`licensing-entitlement-trial.md`](./licensing-entitlement-trial.md) | Licensing, entitlement & trial (snapshot signing, fail-static, offline evaluation, trial anti-abuse) | [`security#1`](https://github.com/Vev-software/security/issues/1) |
