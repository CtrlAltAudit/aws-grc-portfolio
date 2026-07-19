# CloudTrail — Traditional GRC Approach

## What Was Done

Audit logging was configured manually through the AWS Management Console:

1. Created a dedicated S3 bucket (`cloudtrail-logs-<account-id>`) with block public access, versioning, and default encryption enabled
2. Navigated to CloudTrail → Create trail → configured a multi-region trail pointed at that bucket, with log file validation and CloudWatch Logs integration enabled
3. Navigated to CloudWatch → created a metric filter on the CloudTrail log group matching root-account activity, then created an alarm and SNS topic on top of it, and confirmed the email subscription
4. Manually downloaded a `.json.gz` log file from S3, decompressed it, and read through the raw JSON to locate a specific event (a `PutMetricFilter` call) to confirm the pipeline actually produces usable evidence — not just a green checkmark in the console

Total time: roughly 45–60 minutes of console configuration, plus manual file handling to verify the pipeline worked.

Evidence collected: console screenshots and one manually-downloaded, manually-decompressed log file (stored locally — never committed to version control, since raw CloudTrail data contains real account details).

---

## Why This Approach Has Limits

**No repeatability.** If this account were deleted or a new account needed the same baseline, all of these steps — bucket creation, trail configuration, metric filter pattern, alarm thresholds — would need to be manually re-clicked and re-typed, with real risk of a typo in the filter pattern silently breaking the alarm.

**Verification is manual and one-off.** Confirming the pipeline actually works required physically downloading a compressed file, decompressing it outside the console, and reading raw JSON by hand. That's not something a GRC team can realistically do on any kind of recurring schedule — it was done once, here, to prove the concept, not as a repeatable control test.

**Detection depends on someone checking.** The alarm itself is automated, but the underlying configuration — is log file validation still enabled? Is the metric filter pattern still correct? Has anyone since disabled the trail? — is only knowable by going back into the console and looking. There's no diff against a known-good baseline.

**Drift is invisible.** If someone disables log file validation, deletes the metric filter, or changes the S3 bucket's public access settings after this setup, there is no record in version control. The console shows current state only.

**Doesn't scale.** This is workable for one account. At five accounts, replicating the exact metric filter pattern, alarm thresholds, and bucket hardening settings by hand becomes error-prone. At fifty, it's not viable without automation.

---

## The Same Controls, Done Differently

See [`../engineering/`](../engineering/) for the CloudFormation equivalent — the same audit logging baseline deployed as version-controlled, repeatable infrastructure code, with the verification step (finding a real event) turned into a documented, redeployable pattern rather than a one-time manual exercise.
