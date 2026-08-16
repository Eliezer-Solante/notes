![[Pasted image 20260805162530.png]]
![[Pasted image 20260805162549.png]]
![[Pasted image 20260805162719.png]]


**AWS STS (Security Token Service)** is the AWS service that issues **temporary, short-lived security credentials** — instead of long-term access keys — for accessing AWS resources.

### Core Concept

Whenever an IAM role is **assumed** (by a user, EC2 instance, Lambda function, or another AWS account), STS generates a temporary credential set consisting of:

- **Access Key ID**
- **Secret Access Key**
- **Session Token**

These credentials **automatically expire** — typically between **15 minutes and 12 hours**, depending on configuration — reducing the risk if they're ever exposed.

### Why It Matters

STS is the mechanism **behind the scenes** every time a role is assumed. For example, in the lab you just did: once `S3ListRole` was attached to the EC2 instance, STS is what actually issued the temporary credentials the instance used to run `aws s3 ls` — retrieved automatically via the **Instance Metadata Service (IMDS)** at `169.254.169.254`.

### Common STS API Actions

|Action|Use case|
|---|---|
|`AssumeRole`|Assume a role within the same or another AWS account|
|`AssumeRoleWithSAML`|Federated login via SAML (e.g., corporate SSO)|
|`AssumeRoleWithWebIdentity`|Federated login via web identity (Google, Cognito, OIDC)|
|`GetSessionToken`|Add MFA to existing long-term IAM user credentials|
|`GetFederationToken`|Temporary credentials for federated (non-IAM) users|
|`GetCallerIdentity`|Check which identity your current credentials belong to|

### Key Benefit

No long-term credentials to leak, rotate, or forget about — temporary credentials expire automatically, which is why AWS considers **role + STS** the security best practice over static IAM user access keys.

**Simple analogy:** STS is like a hotel issuing a **keycard** that only works for your stay (temporary), instead of giving you a **permanent house key** (long-term access keys) that you'd have to remember to return or that could be copied.