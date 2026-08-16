Here's a comprehensive reference of IAM policy JSON elements with examples for each:

## Basic Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OptionalStatementId",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

---

## 1. `Sid` (Statement ID)

Optional identifier to label a statement — useful for readability when you have multiple statements.

```json
{
  "Sid": "AllowS3ReadAccess"
}
```

> Must be unique within a policy. No spaces allowed in older policy versions (safe to avoid spaces).

---

## 2. `Effect`

Only two possible values: `Allow` or `Deny`.

```json
{ "Effect": "Allow" }
```

```json
{ "Effect": "Deny" }
```

> **Remember:** Explicit Deny always overrides any Allow, regardless of where it comes from.

---

## 3. `Action`

The specific API operation(s) the statement applies to. Can be a single string or an array.

**Single action:**

```json
{ "Action": "s3:GetObject" }
```

**Multiple actions:**

```json
{
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:ListBucket"
  ]
}
```

**Wildcard (all actions in a service):**

```json
{ "Action": "s3:*" }
```

**All actions across all services (dangerous — admin-level):**

```json
{ "Action": "*" }
```

**NotAction (inverse — everything EXCEPT these):**

```json
{ "NotAction": "iam:*" }
```

---

## 4. `Principal` (only used in resource-based policies, e.g., S3 bucket policies, trust policies)

Specifies **who** the statement applies to. Not used in identity-based policies (those are already tied to a user/role).

**Specific IAM user:**

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:user/Bob"
  }
}
```

**Specific IAM role:**

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/MyRole"
  }
}
```

**Entire AWS account:**

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:root"
  }
}
```

**AWS service (e.g., in a role trust policy for EC2/Lambda):**

```json
{
  "Principal": {
    "Service": "ec2.amazonaws.com"
  }
}
```

**Multiple principals:**

```json
{
  "Principal": {
    "AWS": [
      "arn:aws:iam::123456789012:user/Bob",
      "arn:aws:iam::123456789012:role/MyRole"
    ]
  }
}
```

**Everyone (public — use with caution!):**

```json
{ "Principal": "*" }
```

**NotPrincipal (inverse — everyone EXCEPT this):**

```json
{
  "NotPrincipal": {
    "AWS": "arn:aws:iam::123456789012:user/Admin"
  }
}
```

---

## 5. `Resource`

Specifies **what** AWS resource(s) the statement applies to, via ARN.

**Single resource:**

```json
{ "Resource": "arn:aws:s3:::my-bucket" }
```

**Multiple resources:**

```json
{
  "Resource": [
    "arn:aws:s3:::my-bucket",
    "arn:aws:s3:::my-bucket/*"
  ]
}
```

**All resources:**

```json
{ "Resource": "*" }
```

**NotResource (inverse — everything EXCEPT this):**

```json
{
  "NotResource": "arn:aws:s3:::confidential-bucket/*"
}
```

**Dynamic resource using variables (e.g., "own folder" pattern):**

```json
{
  "Resource": "arn:aws:s3:::my-bucket/${aws:username}/*"
}
```

---

## Full Example Combinations

### A. Identity-based policy (attached to a user/role — no Principal needed)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadS3Bucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

### B. Resource-based policy (e.g., S3 bucket policy — Principal required)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificUserAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/Bob"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### C. Trust policy (role assumption — who can assume this role)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2ToAssumeRole",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### D. Explicit Deny statement

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteOnAccountingBucket",
      "Effect": "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:group/accounting"
      },
      "Action": "s3:DeleteObject",
      "Resource": [
        "arn:aws:s3:::accounting1",
        "arn:aws:s3:::accounting1/*"
      ]
    }
  ]
}
```

### E. Permissions boundary example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MaxPermissionsEC2Only",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    }
  ]
}
```

### F. Multi-statement policy (Allow + Deny combined)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFullS3Access",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    },
    {
      "Sid": "DenyDeleteBucket",
      "Effect": "Deny",
      "Action": "s3:DeleteBucket",
      "Resource": "*"
    }
  ]
}
```

> Result: full S3 access **except** deleting buckets — the Deny overrides the broad Allow.

### G. Condition element (bonus — commonly paired with these elements)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyFromOfficeIP",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

---

## Quick Reference Table

|Element|Required?|Used in Identity Policies?|Used in Resource Policies?|Values|
|---|---|---|---|---|
|`Sid`|❌ Optional|✅|✅|Any string, no spaces recommended|
|`Effect`|✅ Required|✅|✅|`Allow` or `Deny`|
|`Action`|✅ Required (or `NotAction`)|✅|✅|`service:Action` string/array|
|`Principal`|✅ Required in resource policies only|❌|✅|AWS account, user, role, service, or `*`|
|`Resource`|✅ Required (or `NotResource`)|✅|✅|ARN string/array or `*`|
Here's a detailed breakdown of the **`Condition`** element — used to add fine-grained, contextual rules to when a policy statement applies.

## Basic Structure

```json
{
  "Condition": {
    "ConditionOperator": {
      "ConditionKey": "ConditionValue"
    }
  }
}
```

Conditions are evaluated as **AND** logic — if there are multiple condition blocks, **all must be true** for the statement to apply. Within a single condition key with multiple values, it's typically **OR** logic (unless using `...IfExists` or special set operators).

---

## 1. String Operators

Used for exact text matching.

```json
"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": "us-east-1"
  }
}
```

|Operator|Meaning|
|---|---|
|`StringEquals`|Exact match|
|`StringNotEquals`|Does not match|
|`StringEqualsIgnoreCase`|Exact match, case-insensitive|
|`StringLike`|Match with wildcards (`*`, `?`)|
|`StringNotLike`|Does not match wildcard pattern|

**Example — StringLike with wildcard:**

```json
"Condition": {
  "StringLike": {
    "s3:prefix": "projects/*"
  }
}
```

---

## 2. Numeric Operators

```json
"Condition": {
  "NumericLessThan": {
    "s3:max-keys": "10"
  }
}
```

|Operator|Meaning|
|---|---|
|`NumericEquals`|Equal to|
|`NumericNotEquals`|Not equal to|
|`NumericLessThan`|Less than|
|`NumericLessThanEquals`|Less than or equal|
|`NumericGreaterThan`|Greater than|
|`NumericGreaterThanEquals`|Greater than or equal|

---

## 3. Date Operators

Great for time-limited access (e.g., temporary contractor access).

```json
"Condition": {
  "DateGreaterThan": {
    "aws:CurrentTime": "2025-01-01T00:00:00Z"
  },
  "DateLessThan": {
    "aws:CurrentTime": "2025-12-31T23:59:59Z"
  }
}
```

|Operator|Meaning|
|---|---|
|`DateEquals`|Exact date/time match|
|`DateNotEquals`|Not equal|
|`DateLessThan`|Before this date|
|`DateLessThanEquals`|On or before|
|`DateGreaterThan`|After this date|
|`DateGreaterThanEquals`|On or after|

---

## 4. Boolean Operator

```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  }
}
```

> Extremely common pattern — **require MFA** before allowing a sensitive action (like deleting resources or modifying IAM).

---

## 5. IP Address Operators

Restrict access based on source IP — commonly used for office/VPN-only access.

```json
"Condition": {
  "IpAddress": {
    "aws:SourceIp": "203.0.113.0/24"
  }
}
```

|Operator|Meaning|
|---|---|
|`IpAddress`|IP is within this CIDR range|
|`NotIpAddress`|IP is NOT within this CIDR range|

---

## 6. ARN Operators

Compare against another ARN — useful for confused-deputy protection or restricting by source resource.

```json
"Condition": {
  "ArnEquals": {
    "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:123456789012:trail/MyTrail"
  }
}
```

|Operator|Meaning|
|---|---|
|`ArnEquals` / `ArnLike`|ARN matches (exact / wildcard)|
|`ArnNotEquals` / `ArnNotLike`|ARN does not match|

---

## 7. Null Operator

Checks whether a condition key exists (or doesn't) in the request.

```json
"Condition": {
  "Null": {
    "aws:TokenIssueTime": "true"
  }
}
```

> `"true"` = key **does not exist**, `"false"` = key **does exist**. Commonly used to distinguish between long-term credentials (no TokenIssueTime) vs. temporary/STS credentials (has TokenIssueTime).

---

## Common Global Condition Keys (usable in any policy)

|Key|Purpose|
|---|---|
|`aws:CurrentTime`|Current date/time|
|`aws:SourceIp`|Requester's IP address|
|`aws:MultiFactorAuthPresent`|Whether MFA was used|
|`aws:PrincipalArn`|ARN of the requesting principal|
|`aws:RequestedRegion`|AWS region the request targets|
|`aws:SourceArn`|ARN of the resource making the request (service-to-service)|
|`aws:username`|IAM username of the requester|
|`aws:TokenIssueTime`|When temporary credentials were issued|
|`s3:prefix`|S3 key prefix (folder-like path)|
|`s3:x-amz-server-side-encryption`|Whether S3 encryption was specified|

---

## Real-World Examples

### A. Require MFA for sensitive actions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllExceptWithMFA",
      "Effect": "Deny",
      "Action": "iam:*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

> Denies all IAM actions unless MFA is present. `BoolIfExists` avoids errors if the key is missing entirely.

### B. Restrict access to office IP only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyFromOffice",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
        }
      }
    }
  ]
}
```

### C. Time-bound access (temporary contractor)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TemporaryAccessWindow",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": {
          "aws:CurrentTime": "2026-08-01T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2026-08-31T23:59:59Z"
        }
      }
    }
  ]
}
```

### D. Restrict S3 access to a specific "folder" matching username (self-service pattern)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUserFolderAccessOnly",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::my-bucket/home/${aws:username}/*"
    }
  ]
}
```

> Note: This uses a **policy variable** (`${aws:username}`) rather than `Condition`, but often used alongside conditions for the same goal.

### E. Enforce encryption on S3 uploads

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### F. Combine multiple conditions (AND logic)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyFromOfficeWithMFADuringBusinessHours",
      "Effect": "Allow",
      "Action": "ec2:TerminateInstances",
      "Resource": "*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2026-01-01T09:00:00Z"
        }
      }
    }
  ]
}
```

> All three conditions must be true simultaneously (office IP **AND** MFA present **AND** after a given date).

---

## `...IfExists` Variant

Every operator has an `...IfExists` version (e.g., `StringEqualsIfExists`, `BoolIfExists`) — this evaluates the condition **only if the key is present** in the request, and is treated as `true` if the key is missing. Useful to avoid unintentionally blocking requests where a condition key isn't applicable.

---

## Quick Reference Table

|Category|Operators|
|---|---|
|String|`StringEquals`, `StringNotEquals`, `StringLike`, `StringNotLike`|
|Numeric|`NumericEquals`, `NumericLessThan`, `NumericGreaterThan`, etc.|
|Date|`DateEquals`, `DateLessThan`, `DateGreaterThan`, etc.|
|Boolean|`Bool`|
|IP Address|`IpAddress`, `NotIpAddress`|
|ARN|`ArnEquals`, `ArnLike`, `ArnNotEquals`, `ArnNotLike`|
|Null (existence check)|`Null`|

**Exam/interview tip:** The most commonly tested condition patterns are **MFA enforcement** (`Bool` + `MultiFactorAuthPresent`), **IP restriction** (`IpAddress` + `SourceIp`), and **encryption enforcement** (`StringNotEquals` + S3 encryption headers) — these show up constantly in real-world security policies and AWS certification exams.



### Common Service Examples

#### S3 (Read-Only)

json

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:ListBucket",
    "s3:GetBucketLocation"
  ],
  "Resource": [
    "arn:aws:s3:::my-bucket",
    "arn:aws:s3:::my-bucket/*"
  ]
}
```

#### EC2 (Read-Only)

json

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:DescribeInstances",
    "ec2:DescribeSecurityGroups",
    "ec2:DescribeVolumes",
    "ec2:DescribeImages"
  ],
  "Resource": "*"
}
```

> Note: EC2 mostly uses `Describe*` instead of `Get*`/`List*` for read operations.

#### IAM (Read-Only)

json

```json
{
  "Effect": "Allow",
  "Action": [
    "iam:GetUser",
    "iam:ListUsers",
    "iam:GetRole",
    "iam:ListRoles",
    "iam:GetPolicy",
    "iam:ListAttachedUserPolicies"
  ],
  "Resource": "*"
}
```

#### DynamoDB (Read-Only)

json

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:DescribeTable"
  ],
  "Resource": "*"
}
```

### S3 Examples

**Write-only:**

json

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:DeleteObject"
  ],
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

**Full access (Read + Write):**

json

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::my-bucket",
    "arn:aws:s3:::my-bucket/*"
  ]
}
```

---

### EC2 Examples

**Write-only (create/manage instances, no describe):**

json

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:RunInstances",
    "ec2:TerminateInstances",
    "ec2:StopInstances",
    "ec2:StartInstances"
  ],
  "Resource": "*"
}
```

> In practice, EC2 write actions almost always need at least `ec2:Describe*` paired with them, since the CLI/console needs to look up resources (AMIs, subnets, security groups) before creating instances.

**Full access:**

json

```json
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*"
}
```

---

### DynamoDB Examples

**Write-only:**

json

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:PutItem",
    "dynamodb:UpdateItem",
    "dynamodb:DeleteItem"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
}
```

**Full access:**

json

```json
{
  "Effect": "Allow",
  "Action": "dynamodb:*",
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
}
```

---

### IAM Examples

**Write-only (manage users, no read/list):**

json

```json
{
  "Effect": "Allow",
  "Action": [
    "iam:CreateUser",
    "iam:DeleteUser",
    "iam:UpdateUser",
    "iam:AttachUserPolicy",
    "iam:DetachUserPolicy"
  ],
  "Resource": "*"
}
```

**Full access:**

json

```json
{
  "Effect": "Allow",
  "Action": "iam:*",
  "Resource": "*"
}
```

---

### AWS Managed Policies (Full Access Examples)

|Policy Name|Scope|
|---|---|
|`AmazonS3FullAccess`|Full S3 read + write|
|`AmazonEC2FullAccess`|Full EC2 read + write|
|`IAMFullAccess`|Full IAM read + write|
|`AmazonDynamoDBFullAccess`|Full DynamoDB read + write|
|`AdministratorAccess`|Full access to **ALL** AWS services (extremely broad — use very cautiously)|
|`PowerUserAccess`|Full access to all services **except IAM/Organizations management**|

**Attach via CLI:**

bash

```bash
aws iam attach-user-policy --user-name USER_NAME --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

---

### Quick Reference Table (All Three Together — S3 example)

|Access Type|Actions|AWS Managed Policy|
|---|---|---|
|**Read-only**|`GetObject`, `ListBucket`|`AmazonS3ReadOnlyAccess`|
|**Write-only**|`PutObject`, `DeleteObject`|_(no direct AWS managed policy — build custom)_|
|**Full (Read+Write)**|`s3:*`|`AmazonS3FullAccess`|