![[Pasted image 20260805142151.png]]
![[Pasted image 20260805142228.png]]

![[Pasted image 20260805142244.png]]
![[Pasted image 20260805142339.png]]
**AWS CloudTrail** is AWS's service for **auditing and logging API activity** across your AWS account — essentially a record of "who did what, when, and from where" for every action taken via the AWS Console, CLI, SDKs, or other AWS services.

**Core concept:**  
Every action in AWS — creating an EC2 instance, deleting an S3 bucket, modifying an IAM policy — is an **API call**. CloudTrail captures these calls as **events** and logs them, giving you a full audit trail of account activity.

**What each event typically records:**

- **Who** — the IAM user, role, or AWS service that made the request
- **What** — the specific API action called (e.g., `s3:DeleteBucket`, `iam:CreateUser`)
- **When** — timestamp of the request
- **Where from** — source IP address
- **Response** — whether the call succeeded or failed, and the response elements

**Types of events CloudTrail logs:**

|Event Type|Description|
|---|---|
|**Management events**|Control plane operations — creating/deleting resources, IAM changes, security group changes. Logged by default.|
|**Data events**|Data plane operations — e.g., S3 object-level activity (`GetObject`, `PutObject`), Lambda function invocations. **Not logged by default** (higher volume, extra cost).|
|**Insight events**|Detects unusual API activity patterns (e.g., a spike in failed calls) using ML-based anomaly detection. Must be enabled separately.|

**Key features:**

- **Event history** — By default, CloudTrail retains the last **90 days** of management events in the console for free, with no setup required.
- **Trails** — To retain logs **longer than 90 days** or for deeper analysis, you create a **Trail**, which delivers events continuously to an **S3 bucket** (and optionally CloudWatch Logs).
- **Multi-region / Organization trails** — A trail can be configured to capture events across **all regions** and even **all accounts in an AWS Organization**, centralizing logs into one S3 bucket.
- **Log file integrity validation** — CloudTrail can generate digitally signed digest files to prove logs haven't been tampered with — critical for compliance/forensics.

**Common use cases:**

1. **Security auditing** — Detect unauthorized access attempts, privilege escalation, or suspicious API calls.
2. **Compliance** — Meet regulatory requirements (HIPAA, PCI-DSS, SOC 2) that mandate activity logging.
3. **Operational troubleshooting** — Determine what change caused an outage (e.g., "who deleted this security group rule?").
4. **Forensics after an incident** — Reconstruct the sequence of actions during a security breach.

**Integration with other services:**

- **CloudWatch Logs** — Stream CloudTrail events to CloudWatch to set up **real-time alarms** (e.g., alert if root account login occurs, or if a security group is modified).
- **Amazon Athena** — Query CloudTrail logs stored in S3 directly using SQL, without needing to load them into a database.
- **AWS Config** — Complements CloudTrail by tracking **resource configuration state over time**, while CloudTrail tracks the **API calls that caused those changes**.

**Best practice example — detecting root account usage:**  
A common security setup is a CloudWatch alarm that triggers whenever CloudTrail logs a `ConsoleLogin` event using the **root** account, since root login should be rare and is a high-risk indicator if unexpected.

**Quick summary table:**

|Feature|Detail|
|---|---|
|Default retention|90 days (event history, free)|
|Long-term storage|Requires a Trail → delivers to S3|
|Logs by default|Management events|
|Extra cost required|Data events, Insight events|
|Common pairing|CloudWatch Logs (alerting), Athena (querying), S3 (storage)|

**Important exam/interview point:** CloudTrail tells you **"what API call was made and by whom"** — it does _not_ by itself tell you the **contents of the resource** or provide real-time blocking. It's a **detective control**, not a preventive one — you use it to observe and audit _after the fact_ (or near real-time via CloudWatch integration), not to stop an action before it happens.
![[Pasted image 20260805142358.png]]


![[Pasted image 20260805142525.png]]