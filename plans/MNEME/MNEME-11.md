---
id: MNEME-11
title: "Port `security_review` as activation"
status: Pending
type: implementation
blocked_by: [MNEME-8]
unlocks: []
confidence: medium
---

## Problem

The `security-review` skill exists as documentation. Port it to a Plexus activation so callers can invoke `security_review.audit` to produce calibrated SOC2 findings.

## Context

The security-review SKILL.md already specifies multi-trial aggregation rules (two-pass review with severity downgrade for single-pass findings) — these map directly onto `swarm.trial` + `swarm.aggregate` with `MajorityEnum` and `MaxSeverity` rules.

## Evidence

`security_review` is the cleanest test of `swarm.aggregate`'s `MajorityEnum` and `MaxSeverity` rules under realistic conditions. It also exercises the multi-trial discipline most directly — the skill's "run two passes, aggregate" instruction becomes literal `swarm.trial(n=2)` + `swarm.aggregate(MajorityEnum)`.

This is the second-best validation skill after `forecast`. Doing it after `forecast` gives us a comparison: does `swarm`-driven multi-trial change findings vs. single-trial?

## Required behavior

`security_review` activation, namespace `security_review`, version `0.1.0`. Methods:

```
security_review.audit(
    codebase_root: String,         // absolute path
    auth_model: String,
    tenant_model: String,
    deployment_topology: String,
    trials: Option<u8>,            // default 2
) -> Stream<AuditEvent>
```

Output (`AuditEvent::Completed`):

| Field | Type |
|-------|------|
| `findings` | Vec<Finding> |
| `summary` | object (counts by severity, counts by control family) |
| `aggregation_metadata` | object (per finding: agreed/single-pass) |

`Finding` shape per the security-review skill's output format (location, control, confidence, gap, impact, evidence, recommendation).

## Acceptance criteria

1. `security_review` activation registered.
2. `mneme run security_review.audit --codebase-root /path --auth-model "JWT" --tenant-model "row-level" --deployment-topology "TLS at ingress" --trials 2` returns findings with calibrated severities.
3. Findings agreed by both trials carry `confidence: high`; single-pass findings carry `confidence: single-pass` and have severity downgraded one level per the skill's rule.
4. Integration test against a known-vulnerable fixture codebase: assert the seeded vulnerability is detected.
5. `cargo build` and `cargo test` pass green.

## Completion

Test passes; status flips to `Complete` in the same commit.
