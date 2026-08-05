# security

> **Visibility:** PUBLIC · **Licence:** CC-BY-4.0
> **Product:** platform · **Owner team:** security

Public security policy, disclosure-process and advisory documentation for VEV
repositories and services.

## Status
Working baseline — repository bootstrap only.

## What this is (and is not)
- **Is:** the public home for VEV security policy, coordinated disclosure process,
  advisories and related documentation.
- **Is not:** secrets, private incident detail, exploit material or internal
  operational runbooks.

## Dependencies
This repo is policy/documentation only. It must remain public and must not rely
on private repositories, feeds or internal-only build steps.

## Quickstart
No site or publishing pipeline is configured yet. The current bootstrap provides
baseline repository governance and disclosure scaffolding.

## Architecture
Repository strategy: ngineering/handbook/02-Repository-Strategy.md §3.1.
Contribution policy: ngineering/handbook/17-Contributing.md.
Repo-local bootstrap ADRs live under docs/adr/.

## Contributing
Security-policy changes should be small, reviewable and precise. Vulnerability
reports themselves do not belong in public pull requests or issues.

## Security
Private disclosure only — see SECURITY.md.