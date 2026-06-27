# IAM — Identity and Access Management Controls

## Overview

This folder contains custom IAM policies designed to demonstrate least privilege principles as they apply to cloud security and GRC engineering. Each policy is scoped to a specific persona and includes explicit Deny statements — not just Allow — to make the controls tamper-resistant.

These policies were created as part of a hands-on AWS GRC practice environment. They are intentionally more restrictive than AWS managed policies to reflect real-world audit and compliance requirements.

---

## Policies

### `auditor-readonly.json`
**Persona:** External auditor or internal audit reviewer

**Design rationale:**
- Grants read access across IAM, S3, CloudTrail, CloudWatch, Config, and EC2 — the services an auditor needs to review access controls and audit logs
- Explicitly **Denies** all write and administrative actions, including creating users, modifying policies, and disabling CloudTrail or Config
- The explicit Deny overrides any other Allow — this ensures the policy is safe to attach even if AWS managed policies are also present

**Control mapping:**
| Control | Framework | Reference |
|---|---|---|
| Least privilege access | CIS AWS Benchmark | 1.16 |
| Audit log read access | SOC 2 | CC7.2 |
| Separation of duties | NIST CSF | PR.AC-4 |

---

### `developer-s3-limited.json`
**Persona:** Application developer who needs S3 access only

**Design rationale:**
- Scoped to a **specific S3 bucket ARN** — not all buckets. Resource-level least privilege, not just action-level
- Allows CloudWatch Logs read so developers can troubleshoot application issues
- Explicitly **Denies** all IAM, CloudTrail, Config, and Organizations actions — a developer has no business touching security infrastructure
- The bucket ARN placeholder (`REPLACE-WITH-YOUR-BUCKET`) is intentional — in practice this would be populated with the actual bucket name at deployment time

**Control mapping:**
| Control | Framework | Reference |
|---|---|---|
| Least privilege access | CIS AWS Benchmark | 1.16 |
| Restrict access to sensitive services | NIST CSF | PR.AC-3 |
| Separation of duties | SOC 2 | CC6.3 |

---

### `admin-with-guardrails.json`
**Persona:** Cloud administrator (day-to-day admin work — not root)

**Design rationale:**
- Grants broad admin access (`Action: *`) but adds two critical Deny guardrails
- **Guardrail 1 — Audit control protection:** Explicitly Denies stopping CloudTrail, disabling Config, deleting GuardDuty, or disabling Security Hub. This prevents an admin (or a compromised admin credential) from disabling the detective controls that would catch malicious activity
- **Guardrail 2 — Region lock:** Denies all actions outside `us-east-1`. This is a common GRC control to prevent resource sprawl and ensure all activity is visible in a single region's audit logs
- In a production environment this policy would be applied via an IAM role, not a user, and would require MFA to assume

**Control mapping:**
| Control | Framework | Reference |
|---|---|---|
| Protect audit logs from modification | CIS AWS Benchmark | 3.2 |
| Restrict AWS region usage | CIS AWS Benchmark | 4.1 |
| Tamper-resistant logging | SOC 2 | CC7.2 |
| Admin access control | NIST CSF | PR.AC-4 |

---

## Evidence

The `evidence/` folder stores locally-generated audit artifacts (IAM Credential Reports, Access Advisor exports). These files are excluded from version control via `.gitignore` because they contain account-specific data. In a real audit engagement these would be stored in a secure evidence management system.
