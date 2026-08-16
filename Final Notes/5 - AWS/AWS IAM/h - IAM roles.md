![[Pasted image 20260805135614.png]]
![[Pasted image 20260805135720.png]]

**IAM Roles** are AWS identities with specific permissions that can be **temporarily assumed** by trusted users, applications, or AWS services — instead of being tied permanently to one person like an IAM user.

**Core characteristics:**

- **No long-term credentials** — Roles don't have a password or permanent access keys. Instead, when something "assumes" a role, AWS's **STS (Security Token Service)** issues **temporary security credentials** that automatically expire (typically within 15 minutes to 12 hours).
- **Requires a Trust Policy** — Every role has a trust policy (also called an "assume role policy") that defines **who or what is allowed to assume it**. This could be:
    - An IAM user or role in the same or a different AWS account
    - An AWS service (like EC2, Lambda, ECS)
    - A federated identity (e.g., via SAML, Google, or Amazon Cognito)
- **Permissions Policy** — Just like users, roles also have permissions policies attached that define **what actions they can perform** once assumed.

**Common use cases:**

1. **AWS services needing access to other services**  
    Example: An EC2 instance needs to read from an S3 bucket → attach a role to the EC2 instance instead of hardcoding access keys.
2. **Cross-account access**  
    Example: Allowing a user in Account A to access resources in Account B without creating a duplicate user.
3. **Federated / SSO access**  
    Example: Employees log in via corporate identity provider (Okta, Azure AD) and assume a role to get temporary AWS access — no IAM user needed at all.
4. **Lambda functions / ECS tasks**  
    Example: A Lambda function assumes an execution role to write logs to CloudWatch or read from DynamoDB.

**How assuming a role works (simplified flow):**

1. A trusted entity requests to assume the role via `sts:AssumeRole`.
2. AWS checks the role's **trust policy** to confirm the entity is allowed.
3. If permitted, STS issues **temporary credentials** (access key, secret key, session token).
4. The entity uses these temporary credentials to act with the role's permissions until they expire.

**Why roles are considered best practice:**  
Because credentials are temporary and auto-expiring, roles significantly reduce security risk compared to long-lived IAM user access keys — there's nothing permanent to leak or forget to rotate.

**Quick summary table:**

|Feature|IAM Role|
|---|---|
|Credentials|Temporary (via STS)|
|Used by|Users, services, or other accounts|
|Requires trust policy|✅ Yes|
|Ideal for|EC2/Lambda access, cross-account access, federation|

![[Pasted image 20260805135834.png]]
![[Pasted image 20260805135903.png]]

![[Pasted image 20260805135952.png]]
![[Pasted image 20260805140008.png]]


![[Pasted image 20260805140025.png]]

