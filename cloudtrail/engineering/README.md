# CloudTrail — GRC Engineering Approach

## What This Does Differently

The same audit logging baseline from the traditional approach is expressed here as a CloudFormation template: [`cloudformation/cloudtrail-baseline.yaml`](cloudformation/cloudtrail-baseline.yaml).

A single command deploys the entire baseline to any AWS account:

```bash
aws cloudformation deploy \
  --template-file cloudtrail/engineering/cloudformation/cloudtrail-baseline.yaml \
  --stack-name cloudtrail-baseline \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      NotificationEmail=<your-email>
```

Total time: under 2 minutes. After deploying, check the notification email inbox for an SNS subscription confirmation — CloudFormation can create the subscription, but AWS requires a human to click the confirmation link before email alerts actually deliver (there's no way to automate that step; it's a deliberate anti-spam control on AWS's side).

---

## What Gets Deployed

- **A dedicated, hardened S3 bucket** (`CloudTrailBucket`) — versioned, encrypted, all public access blocked, with a bucket policy scoped so only the CloudTrail service principal can write to it, only under its own `AWSLogs/<account-id>/` prefix, and only when the request originates from this specific trail's ARN (not just any trail in the account)
- **A multi-region trail** (`ManagementEventsTrail`) — captures all management events (Read + Write), with log file validation enabled
- **A CloudWatch Logs integration** — a narrowly-scoped IAM role lets CloudTrail stream events in near-real-time to a dedicated log group, nothing else
- **A metric filter, alarm, and SNS topic** — parses the live log stream for root-account activity and emails an alert within minutes

---

## Why This Approach Is Better for GRC

**Repeatable.** The exact metric filter pattern that catches root logins — easy to get subtly wrong by hand — is defined once, in code, and deployed identically every time. No risk of a typo silently breaking detection in one account but not another.

**Verifiable without manual file handling.** In the traditional path, confirming the pipeline worked required downloading and decompressing a log file by hand. Here, the same verification — "does this deployment actually produce a findable, attributable event" — is a repeatable pattern: deploy, trigger an action, query CloudTrail Event history or `aws logs filter-log-events` via the CLI. See [`../../iam/engineering/README.md`](../../iam/engineering/README.md) for the same principle applied to Project 1's permissions boundary control test.

**The template IS the evidence.** An auditor doesn't need to trust that log file validation is "probably still on" — the template in Git is the authoritative record of intended configuration, and the deployed stack is queryable via API to confirm actual state matches.

**Drift-detectable.** If someone disables log file validation or loosens the bucket policy outside of CloudFormation, drift detection catches it — the same mechanism demonstrated in Project 1.

---

## Security Review Findings and Remediation

A security review of this template (same rigor as Project 1's review) found one moderate and one minor item. One was fixed; the other is a documented, deliberate tradeoff rather than a silent gap:

**Fixed — bucket policy `aws:SourceArn` scoping.** The initial version scoped the bucket policy using only an `aws:SourceAccount` condition, which blocks cross-account writes but leaves a narrow gap: any other CloudTrail trail created in this same account (by anyone with `cloudtrail:CreateTrail` permission) could also write into this bucket. AWS's own reference bucket policies additionally scope with `aws:SourceArn` to the specific trail ARN — but doing that with `!GetAtt ManagementEventsTrail.Arn` would create a circular dependency, since the trail also depends on this bucket policy existing first. The fix: the trail's ARN is constructed as a deterministic string (`region` + `account ID` + the trail's own parameterized name) rather than referenced via the resource itself — same precision, no circular dependency.

**Documented tradeoff — SSE-S3 instead of SSE-KMS.** The bucket uses SSE-S3 (AES-256) encryption, not a customer-managed KMS key. A KMS key would add independent access logging on key usage and the ability to revoke decrypt access without touching the bucket policy — a real hardening improvement for a production audit-log bucket. It's deliberately not used here because KMS keys cost ~$1/month and this project is scoped to stay within AWS free tier. SSE-S3 still satisfies CIS 2.1.5/3.3 and the SOC 2/NIST CSF controls mapped below; the gap is between "compliant" and "maximally hardened," not between "compliant" and "non-compliant."

## Known Limitation: Email Subscription Confirmation

CloudFormation can create the SNS topic and subscription, but AWS will not deliver alerts until a human clicks the confirmation link AWS emails to the subscribed address. This is not a gap in the template — it's a deliberate AWS safeguard against using SNS to spam arbitrary email addresses — but it does mean "stack deployed successfully" is not the same as "alerting is actually live." The verification checklist below treats subscription confirmation as a separate, required step.

---

## Control Mapping

Framework note: ISO 27001:2022 splits logging into two distinct controls not separated in the 2013 revision — **A.8.15 (Logging)** covers the existence and protection of logs, **A.8.16 (Monitoring activities)** covers actively watching them. This project's alarm maps specifically to A.8.16; the trail and log file validation map to A.8.15.

| Activity | CIS Benchmark | SOC 2 | NIST CSF | CMMC 2.0 | ISO 27001:2022 | ISO 27017 |
|---|---|---|---|---|---|---|
| CloudTrail enabled, multi-region, management events (R+W) | 3.1 | CC7.2 | DE.CM-3 | AU.L2-3.3.1 | A.8.15 | CLD.12.4.1 |
| Log file validation (SHA-256 hash chain) | 3.2 | CC7.2 | PR.DS-6 | AU.L2-3.3.8 | A.8.15 | CLD.12.4.1 |
| S3 log bucket — block public access, versioning, encryption | 2.1.5, 3.3 | CC6.1 | PR.AC-3, PR.DS-1 | AC.L1-3.1.1, SC.L2-3.13.11 | A.8.10, A.8.24 | CLD.9.5.1 |
| CloudWatch Logs integration (real-time streaming) | 3.4 | CC7.2 | DE.AE-3 | AU.L2-3.3.2 | A.8.16 | CLD.12.4.1 |
| Root account login alarm (metric filter + SNS) | 3.3 | CC7.3 | DE.CM-3 | AU.L2-3.3.1 | A.8.16 | CLD.12.4.1 |

**NIST CSF framing:** this project sits almost entirely in the **Detect** function — in contrast to [Project 1 (IAM)](../../iam/), which was mostly **Protect**/preventive (Deny statements, least privilege, permissions boundaries). Together they demonstrate two different control types: stopping bad actions before they happen versus noticing and proving they happened after the fact. See the root portfolio README for how this pattern continues across the remaining projects.
