# Control — Identity Isolation

**Control area:** Identities
**Applies to:** Veritas (Tier 1)
**Owner:** Compliance Engineering

## Statement

Veritas provisions exactly two execution roles, `veritas-ingestor` and
`veritas-reporter`. Neither role may be assumed by a principal belonging to
another application, and no role belonging to another application may be bound
to a Veritas resource.

## Why this control exists

Tier 1 classification is inherited through data, not through function. Veritas
is a reporting system, but it reads trade execution records that originate in
Tier 1 systems, so it carries Tier 1 controls. A shared identity would let a
lower-tier application reach Tier 1 data without ever being classified for it,
which defeats the classification entirely.

## How it is verified

Every change to either role's policy requires a Compliance Engineering reviewer
on the pull request before merge. The reviewer checks three things:

1. The change does not add a trust relationship to a principal outside Veritas.
2. The change does not widen a resource ARN to a wildcard.
3. The change is justified in the pull request description.

Approval by Platform Engineering alone does not satisfy this control.

## Failure mode

If this control is not held, the first symptom is usually an access-review
finding rather than an incident: a quarterly review discovers a role binding
nobody can account for. By then the data has already been reachable for up to a
quarter, which is why the pull-request gate is the control and the access review
is only the backstop.
