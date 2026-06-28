# IAM — Traditional GRC Approach

## What Was Done

The IAM environment in this account was configured manually through the AWS Management Console:

1. Navigated to IAM → Account settings → edited password policy fields one by one
2. Navigated to IAM → Users → Create user for each of three personas (auditor, admin, developer)
3. Navigated to IAM → Policies → Create policy → pasted JSON → named each policy
4. Navigated back to each user → Add permissions → attached the corresponding policy
5. Logged in as `bryanne-auditor` in an incognito window to verify the deny worked
6. Downloaded a Credential Report as a CSV from IAM → Credential report

Total time: ~45 minutes of console clicks across multiple screens.

Evidence collected: screenshots of each configuration screen, credential report CSV (stored locally — never committed to version control because it contains account data).

---

## Why This Approach Has Limits

**No repeatability.** If this account were deleted or a new account needed the same baseline, every step would have to be repeated manually. There is no executable artifact — only a human's memory of what was clicked.

**Evidence is a screenshot.** Screenshots confirm a state existed at a point in time but can't be queried, diffed, or verified programmatically. An auditor receiving a screenshot has to trust it wasn't taken before or after the real configuration.

**Drift is invisible.** If someone changes the password policy or attaches a new policy to a user after this setup, there is no record in version control. The console shows current state only — not history.

**Doesn't scale.** This approach works for one account. At five accounts it becomes error-prone. At fifty accounts it becomes impossible without automation.

**GRC-team-centric.** The process lives entirely in the GRC/security team's head. Developers can't review or propose changes to policies the way they would a pull request. There's no peer review workflow.

---

## The Same Controls, Done Differently

See [`../engineering/`](../engineering/) for the CloudFormation equivalent — the same IAM baseline deployed as version-controlled, repeatable infrastructure code.

The controls are identical. The difference is the artifact: a screenshot vs. a template.
