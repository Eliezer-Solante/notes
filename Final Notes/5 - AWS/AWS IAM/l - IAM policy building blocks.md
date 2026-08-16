![[Pasted image 20260805152236.png]]

![[Pasted image 20260805152354.png]]
This is an example resource policy demonstrating a **combined IP + time-based access restriction** using the `Condition` element. Let's break it down:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": [
            "trusted-ip-range-1",
            "trusted-ip-range-2"
          ]
        },
        "NumericLessThan": {
          "aws:CurrentTime": "09:00"
        },
        "NumericGreaterThan": {
          "aws:CurrentTime": "17:00"
        }
      }
    }
  ]
}
```

## Line-by-line breakdown

- **`"Effect": "Deny"`** — This statement blocks access (rather than granting it).
- **`"Action": "*"`** — Applies to **all actions**, across all services. 
- **`"Resource": "*"`** — Applies to **all resources**.
- **`"NotIpAddress"` + `"aws:SourceIp"`** — This is the inverse of `IpAddress`. It evaluates to **true** when the requester's IP is **NOT** in the listed trusted ranges (`trusted-ip-range-1`, `trusted-ip-range-2`).
- **`"NumericLessThan"` + `"aws:CurrentTime": "09:00"`** — True when the current time is **before 9:00 AM**.
- **`"NumericGreaterThan"` + `"aws:CurrentTime": "17:00"`** — True when the current time is **after 5:00 PM**.
## What this policy actually does

Since all conditions inside a single `Condition` block are combined with **AND** logic, this statement only triggers the Deny when:

1. The request comes from an IP **outside** the trusted ranges, **AND**
2. The current time is before 9:00 AM, **AND**
3. The current time is after 5:00 PM

⚠️ **Important catch:** Conditions #2 and #3 **cannot both be true at the same time** — a single moment in time can't be simultaneously before 9:00 AM _and_ after 5:00 PM. Since this uses **AND** logic (not OR), as written, this Deny statement would **never actually trigger** — it's logically impossible to satisfy all three conditions simultaneously.

## What was likely intended

The diagram is illustrating the _concept_ of combining IP + time restrictions, but the time logic as written is a common mistake. To correctly express **"deny access outside business hours (9 AM–5 PM) from untrusted IPs,"** you'd need **OR** logic between the two time conditions, which IAM condition blocks can't do directly with the same operator. The typical real-world fix is to split it into **two separate Deny statements**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideBusinessHoursMorning",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": ["trusted-ip-range-1", "trusted-ip-range-2"]
        },
        "DateLessThan": {
          "aws:CurrentTime": "09:00:00Z"
        }
      }
    },
    {
      "Sid": "DenyOutsideBusinessHoursEvening",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": ["trusted-ip-range-1", "trusted-ip-range-2"]
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "17:00:00Z"
        }
      }
    }
  ]
}
```

> With two separate statements, if **either** one evaluates true, access is denied — correctly blocking access before 9 AM **or** after 5 PM from untrusted IPs.

## What the concept is teaching

Despite the logic slip in the example, the core lesson is valuable and commonly tested:

- You can **stack multiple condition operators** in one statement (all combined with AND).
- **`NotIpAddress`** is the negation of `IpAddress` — useful for "deny unless from trusted source" patterns.
- Combining **network-based** (`SourceIp`) and **time-based** (`CurrentTime`) conditions is a real-world pattern for enforcing "only accessible from the office, during business hours."
- Also note: `aws:CurrentTime` expects a full **ISO 8601 timestamp** (e.g., `2026-08-05T09:00:00Z`), not just `"09:00"` — the shorthand times in the diagram are simplified for illustration and wouldn't work as literal AWS syntax.



![[Pasted image 20260805152439.png]]

Here's a clean summary of the lab for your notes:

# Lab Notes: EC2 Instance Role Access to S3

## Objective

Demonstrate how an EC2 instance can access AWS services (S3) securely using an **IAM Role** instead of hardcoded credentials — and observe the difference in access **before** and **after** attaching the role.

## Region

`us-east-1`

---

## Steps Performed

**1. Created S3 Bucket**

- Bucket with prefix `kk-resource-` created (target resource for read access testing).

**2. Created IAM Role**

- Role name: `S3ListRole`
- Trusted entity: **EC2** (allows EC2 service to assume this role)
- Permission policy attached: `AmazonS3ReadOnlyAccess` (AWS managed policy)

**3. Created SSH Key Pair**

- Generated key pair (RSA, no email comment) using:
```bash
  ssh-keygen -t rsa -b 4096
```
- Saved as `Ec2instancekeypair` / `Ec2instancekeypair.pub`
- Imported the **public key** (`.pub` file) into AWS Console → EC2 → Key Pairs, using the same name `Ec2instancekeypair`, so it could be selected during instance launch.

**4. Launched EC2 Instance**

- Instance name: `S3InstanceRoleTest`
- AMI: Amazon Linux (free tier)
- Instance type: `t2.micro`
- Key pair: `Ec2instancekeypair`
- Security group: SSH (port 22) open for public access
- **No IAM role attached at launch** (intentional — tested later)

**5. Connected via SSH & Tested S3 Access (before role attached)**

```bash
chmod 400 /root/.ssh/Ec2instancekeypair
ssh -i "/root/.ssh/Ec2instancekeypair" ec2-user@<Public-DNS>
aws s3 ls
```

- **Expected result:** Access denied / unable to list buckets — since the instance has no credentials or IAM role attached yet.

**6. Attached IAM Role to Running Instance**

- EC2 Console → select instance → **Actions → Security → Modify IAM Role**
- Selected `S3ListRole` → clicked **Update IAM Role** (attach)

**7. Re-tested S3 Access (after role attached)**

```bash
aws s3 ls
```

- **Expected result:** Successfully lists S3 buckets — since the instance now assumes `S3ListRole`, which grants `AmazonS3ReadOnlyAccess`.

---

## Key Takeaways / Concepts Demonstrated

|Concept|Demonstrated by|
|---|---|
|**IAM Roles for EC2**|Role trust policy allows EC2 service to assume it|
|**No hardcoded credentials needed**|`aws s3 ls` worked without configuring access keys on the instance|
|**Roles can be attached/modified after launch**|IAM role wasn't required at instance creation — added later via console|
|**Least privilege**|`AmazonS3ReadOnlyAccess` only grants read access, not write/delete|
|**Instance metadata service (IMDS)**|Under the hood, the EC2 instance retrieves temporary STS credentials automatically via `169.254.169.254` once a role is attached|
|**SSH key pair workflow**|Public key generated externally and imported into AWS, rather than AWS-generated `.pem`|

**Core lesson:** Attaching an IAM role to an EC2 instance is the **secure, best-practice way** to grant AWS service permissions to applications running on the instance — avoiding the security risk of storing long-term access keys directly on the server.