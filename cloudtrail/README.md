# CloudTrail — Audit Logging

*Project 2 of 6 — AWS GRC Engineering Portfolio*

## Overview

This folder demonstrates AWS CloudTrail audit logging built two ways — a traditional manual console setup and a CloudFormation engineering path. Where [Project 1 (IAM)](../iam/) was primarily a **preventive** control (least privilege, explicit denies), this project is primarily a **detective** control: it doesn't stop anything, it builds the capability to notice what happened, prove it happened, and alert on it in near real time.

| | Traditional GRC | GRC Engineering |
|---|---|---|
| **How controls are deployed** | Console clicks, manual | CloudFormation template |
| **Evidence** | Screenshots, manually downloaded log files | Deployed stack + queryable CloudTrail API |
| **Repeatability** | Redo manually each time | Same command, any account |
| **Detection latency** | Discovered at next periodic review | Near real-time via CloudWatch alarm |
| **Log integrity proof** | Trust the console's word | SHA-256 log file validation, verifiable on demand |
| **Scales to 50 accounts** | No | Yes |

---

## Folder Structure

```
cloudtrail/
├── traditional/
│   ├── README.md          ← what was done manually and why it has limits
│   └── evidence/           ← local only (gitignored) — screenshots, downloaded log files
└── engineering/
    ├── README.md            ← the automated approach and its advantages
    ├── cloudformation/
    │   └── cloudtrail-baseline.yaml   ← deploys the full baseline in one command
    └── screenshots/           ← redacted evidence embedded in the engineering README
```

---

## What This Deploys

- A dedicated, hardened S3 bucket for log storage (block public access, versioning, encryption)
- A multi-region CloudTrail trail capturing management events (Read + Write) with **log file validation** enabled — a SHA-256 hash chain that proves logs haven't been tampered with after the fact
- Real-time streaming to CloudWatch Logs
- A metric filter + CloudWatch Alarm + SNS topic that fires the moment the root account is used — CIS AWS Benchmark 3.3

See [`traditional/README.md`](traditional/README.md) and [`engineering/README.md`](engineering/README.md) for the two approaches in detail.
