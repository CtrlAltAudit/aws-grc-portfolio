# IAM — Identity and Access Management

*Project 1 of 6 — AWS GRC Engineering Portfolio*

## Overview

This folder demonstrates the same IAM controls implemented two ways — the traditional GRC approach and the GRC engineering approach. The controls are identical. The difference is the artifact each approach produces and what that means for repeatability, evidence quality, and scale.

| | Traditional GRC | GRC Engineering |
|---|---|---|
| **How controls are deployed** | Console clicks, manual | CloudFormation template |
| **Evidence** | Screenshots, credential report CSV | Deployed stack + Git history |
| **Repeatability** | Redo manually each time | Same command, any account |
| **Change tracking** | None (console has no history) | Git commits with author + rationale |
| **Peer review** | Email or ticket | Pull request |
| **Scales to 50 accounts** | No | Yes |

---

## Folder Structure

```
iam/
├── traditional/
│   ├── README.md          ← what was done manually and why it has limits
│   └── evidence/          ← local only (gitignored) — credential reports, screenshots
├── engineering/
│   ├── README.md          ← the automated approach and its advantages
│   └── cloudformation/
│       └── iam-baseline.yaml   ← deploys the full baseline in one command
└── policies/
    ├── auditor-readonly.json        ← scoped read-only with explicit deny on writes
    ├── developer-s3-limited.json    ← S3-only access scoped to a specific bucket
    └── admin-with-guardrails.json   ← admin access with tamper-resistant audit controls
```

The `policies/` folder holds the source-of-truth policy logic. In the traditional path they were created by pasting this JSON into the console. In the engineering path the same Allow/Deny logic is re-expressed inline in the CloudFormation template (not a direct file reference, since CFN inlines policy documents) — the two are kept in sync by hand.

---

## The Policies

### `auditor-readonly`
**Persona:** External auditor or internal audit reviewer

Grants read access across IAM, S3, CloudTrail, CloudWatch, Config, and EC2. Explicitly Denies all write and administrative actions. The explicit Deny overrides any other Allow — this makes the policy safe to attach alongside other policies without risk of privilege escalation.

Maps to: CIS 1.16 | SOC 2 CC6.3 | NIST CSF PR.AC-4 | CMMC AC.L1-3.1.1, AC.L2-3.1.5 | ISO 27001 A.5.15, A.5.3 | ISO 27017 CLD.9.2.3, CLD.6.3.1

### `developer-s3-limited`
**Persona:** Application developer

Scoped to a specific S3 bucket ARN — not all buckets. Resource-level least privilege, not just action-level. Explicitly Denies all IAM, CloudTrail, Config, and Organizations actions.

Maps to: CIS 1.16 | SOC 2 CC6.3 | NIST CSF PR.AC-4 | CMMC AC.L1-3.1.1, AC.L1-3.1.2 | ISO 27001 A.5.15, A.5.18 | ISO 27017 CLD.9.2.3

### `admin-with-guardrails`
**Persona:** Cloud administrator (day-to-day work — not root)

Grants broad admin access but with two Deny guardrails: (1) cannot disable CloudTrail, Config, GuardDuty, or Security Hub — detective controls resist casual or accidental tampering; (2) cannot act outside us-east-1 — region lock keeps all activity visible in one audit log.

**Mitigated via permissions boundary (engineering path only).** The identity-based Deny above can, on its own, be detached or deleted by an admin with `iam:*` — it protects against accidental or scripted misuse, not a determined or fully compromised credential. The CloudFormation-deployed admin user closes this specific gap with a **permissions boundary** (`AdminPermissionsBoundary`) that denies detaching/deleting the guardrail policy and denies removing or swapping the boundary itself. See [`engineering/README.md`](engineering/README.md#permissions-boundary-closing-the-self-escalation-gap) for details.

**Residual gap:** a permissions boundary is still an in-account, identity-scoped control — an admin with `iam:*` could still delete the *boundary policy itself* (CloudFormation self-references prevented adding that specific denial without a circular dependency) or, in a multi-account setup, this whole mechanism is superseded by an AWS Organizations Service Control Policy, which sits outside any single account's IAM and can't be self-modified at all. The boundary meaningfully raises the bar; it isn't the final word.

Maps to: CIS 3.2, 4.1 | SOC 2 CC7.2, CC6.6 | NIST CSF PR.PT-1, PR.AC-5 | CMMC AU.L2-3.3.1, CM.L2-3.4.6 | ISO 27001 A.8.15, A.8.19 | ISO 27017 CLD.12.4.5, CLD.12.1.5
