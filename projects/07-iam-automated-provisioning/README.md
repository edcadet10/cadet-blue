# 07 · Automated Joiner/Mover/Leaver (Microsoft Graph)

![Domain](https://img.shields.io/badge/Domain-IAM%20%2F%20Automation-1f6feb)
![Cert](https://img.shields.io/badge/Maps%20to-Security%2B-E2231A)
![Status](https://img.shields.io/badge/Status-Planned-lightgrey)

> Automate the identity lifecycle (joiner / mover / leaver) against Entra ID using the Microsoft
> Graph API, with a least-privilege identity.

## Objective

Show identity automation: turning manual onboarding/offboarding into repeatable, least-privilege
Graph API operations — the kind of glue that keeps IAM consistent and auditable.

## Based on real work

Extends a **live action API** (`helpdesk-actions-api`) already built as the tool layer for an
Azure AI help-desk agent (Project 10).

## Planned scope

- Microsoft Graph operations: create/disable users, group membership, password reset
- **Least-privilege** app registration / managed identity (only the Graph scopes needed)
- Joiner/mover/leaver flows with logging for audit
- Safe-by-default: dry-run mode before any write

## Skills to demonstrate

Microsoft Graph API · identity lifecycle automation · least-privilege app permissions · managed identity · auditable operations

---
*Status: Planned (new build, extends real work). Part of the [cadet-blue portfolio](../../README.md).*
