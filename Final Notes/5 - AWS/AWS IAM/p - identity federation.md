![[Pasted image 20260805163139.png]]

This diagram illustrates the **end-to-end identity federation flow** — exactly the workforce/SAML federation pattern we just discussed, now shown step-by-step.

## The 5-Step Federation Flow

**1. Sign in with Credentials** The user authenticates directly with their **Organization's IdP** (e.g., Active Directory, Okta, Azure AD) — using their existing corporate username/password. AWS is not involved yet.

**2. Receive auth response** The IdP verifies the credentials and sends back an **authentication response** (e.g., a SAML assertion or OIDC token) confirming the user's identity.

**3. API call to assume role** The user (or their client application) takes that auth response and calls **AWS STS** — specifically `sts:AssumeRoleWithSAML` (or `AssumeRoleWithWebIdentity` for OIDC) — presenting the IdP's token as proof of identity.

**4. Return temporary credentials** STS validates the token against the **trust relationship** already configured between AWS and the IdP, then issues **temporary security credentials** (Access Key, Secret Key, Session Token) — auto-expiring, just like we covered earlier.

**5. Use credentials to access AWS** The user now uses those temporary credentials to directly access **AWS Resources**, with permissions defined by whatever IAM role was assumed.

## The Critical Piece: "Federated Trust"

Notice the arrow labeled **"Federated Trust"** connecting the **Organization's IdP** directly to **AWS**. This represents the **trust relationship** that must be pre-configured (usually by an AWS admin) — essentially telling AWS: _"Trust authentication responses coming from this specific IdP."_

This trust is what's actually configured in the IAM Role's **trust policy** — for example:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:saml-provider/MyCorpIdP"
  },
  "Action": "sts:AssumeRoleWithSAML",
  "Condition": {
    "StringEquals": {
      "SAML:aud": "https://signin.aws.amazon.com/saml"
    }
  }
}
```

## Why This Matters (Key Takeaway)

Notice that **AWS never sees or stores the user's actual password** — the entire authentication happens with the Organization's IdP (steps 1–2). AWS only ever receives a **signed token/assertion** as proof, and in exchange issues **temporary** credentials via STS. This is the core security benefit of federation: **centralized authentication, decentralized (temporary) access**.

## Mapping to Concepts We've Covered

|Diagram Step|AWS Concept|
|---|---|
|Steps 1–2|External IdP authentication (SAML/OIDC)|
|Step 3|`sts:AssumeRoleWithSAML` API call|
|Step 4|**STS** issuing temporary credentials (auto-expiring)|
|Step 5|Access governed by the **IAM Role's** permissions policy|
|"Federated Trust" arrow|The **trust policy** on the IAM Role, specifying the IdP as a trusted principal|

## Simple Analogy (extending the earlier one)

This is like using your **company badge** (corporate credentials) to check in at your company's **security desk** (IdP) first, who then calls ahead to the **building next door** (AWS) to vouch for you — that building then issues you a **temporary visitor pass** (STS credentials) that expires at the end of the day, rather than giving you your own permanent key to their building.


![[Pasted image 20260805163534.png]]
This diagram highlights the **three core federation protocols/standards that AWS supports** for identity federation — essentially the "connectors" that let external identities talk to AWS.

## The Three Standards

### 1. SAML 2.0 (Security Assertion Markup Language)

- **Primary use:** Enterprise/workforce federation
- Used when employees authenticate via a corporate IdP (Active Directory via AD FS, Okta, Azure AD, Ping Identity)
- The IdP issues a **signed XML assertion** confirming the user's identity and attributes
- AWS API: `sts:AssumeRoleWithSAML`
- This is exactly the flow shown in the previous diagram (Organization's IdP → AWS)

### 2. OpenID Connect (OIDC)

- **Primary use:** Modern federation for both workforce and web/mobile apps
- Built **on top of OAuth 2.0**, but adds an **identity layer** — meaning it doesn't just authorize access to resources, it also verifies **who the user is** (via an ID Token, typically a JWT)
- Common providers: Google, Amazon Cognito, Azure AD (also supports OIDC), GitHub Actions (for CI/CD federation)
- AWS API: `sts:AssumeRoleWithWebIdentity`
- Increasingly the **modern standard** replacing older SAML setups where possible, since it's simpler and more widely supported by modern IdPs

### 3. OAuth 2.0

- **Primary use:** Authorization framework (not authentication by itself)
- OAuth 2.0 handles **granting access to resources** on behalf of a user (e.g., "allow this app to read my Google Drive files") — but it doesn't inherently verify identity
- This is why **OIDC exists** — it's built as an identity layer _on top of_ OAuth 2.0 to add proper authentication
- In AWS federation, you'll rarely use "raw" OAuth 2.0 alone — it's almost always paired with OIDC for identity verification

## Important Distinction: OAuth 2.0 vs OIDC

![[Pasted image 20260805163623.png]]

> This is a very commonly tested distinction: **OAuth 2.0 authorizes, OIDC authenticates.**

## How This Maps to AWS Federation Options

|Protocol|AWS Mechanism|Typical Use Case|
|---|---|---|
|SAML 2.0|`AssumeRoleWithSAML`, IAM Identity Center|Enterprise employee SSO|
|OpenID Connect|`AssumeRoleWithWebIdentity`, Cognito, IAM OIDC provider|Web/mobile apps, GitHub Actions/CI-CD, Kubernetes (EKS IRSA)|
|OAuth 2.0|(Underlying framework for OIDC)|Rarely used alone with AWS directly|

## Connecting to Earlier Concepts

This diagram is really just **naming the specific protocols** behind the two federation types we discussed earlier:

- **Workforce federation** → typically **SAML 2.0**
- **Web/mobile federation** → typically **OIDC** (via Cognito or a custom OIDC identity provider)

**Real-world modern example:** GitHub Actions using **OIDC** to assume an AWS IAM role for CI/CD deployments — no long-term AWS access keys stored in GitHub secrets at all, since GitHub acts as the OIDC identity provider and AWS trusts tokens it issues.

![[Pasted image 20260805163643.png]]
This diagram shows the **customer-facing federation flow** — the exact OIDC pattern we just discussed, now applied to public identity providers like Facebook, Amazon, Google, and Apple.

## The 5-Step Web Identity Federation Flow

**Step 1 — User signs in** The **User** interacts with the **Browser**, which redirects them to sign in with one of the public identity providers (Facebook, Amazon, Google, or Apple).

**Step 2 — IdP returns identity token** The chosen provider authenticates the user and sends back an **identity/OIDC token** (a signed JWT) to the browser, confirming who the user is.

**Step 3 — Browser calls STS with the token** The browser sends that token to **AWS STS**, calling `sts:AssumeRoleWithWebIdentity` — presenting the IdP's token as proof of identity.

**Step 4 — STS returns temporary credentials** STS validates the token against the trust configured for that identity provider, then issues **temporary security credentials** (Access Key, Secret Key, Session Token) — scoped to whatever IAM role is tied to that provider.

**Step 5 — Browser accesses AWS Resources directly** Using those temporary credentials, the browser can now directly access **AWS Resources** (shown here as S3, DynamoDB, and other services) — **without any backend server in between**.

## Key Insight: No Backend Server Required

Notice this flow happens **entirely client-side** (User → Browser → AWS) — there's no application server acting as a middleman handling credentials. This is the classic architecture for **serverless / mobile / single-page apps**, where the browser or mobile app talks to AWS resources directly using scoped, temporary permissions.

## Why Not Use IAM Users for App Customers?

Imagine a mobile app with 1 million users — creating 1 million IAM users would be:

- Impossible to manage at scale
- A massive security liability (long-term credentials embedded in app code)

Instead, **each user authenticates via a provider they already trust** (Google, Facebook, etc.), and AWS grants **temporary, scoped access** — solving both problems at once.

## Connecting to Previous Diagrams

|This Diagram|Earlier SAML Diagram|
|---|---|
|Facebook / Google / Apple / Amazon|Organization's IdP|
|`AssumeRoleWithWebIdentity`|`AssumeRoleWithSAML`|
|OIDC token|SAML assertion|
|Used by: **customers/end users**|Used by: **employees**|
|Common pairing: **Amazon Cognito** (manages this flow for you)|Common pairing: **IAM Identity Center**|

## Practical Note: Cognito Usually Sits in the Middle

In real-world implementations, this flow is rarely built by hand — **Amazon Cognito Identity Pools** typically handles steps 1–4 automatically, managing the exchange between the social IdP token and STS temporary credentials, so developers don't need to implement the raw OIDC handshake themselves.

## Simple Analogy

This is like using your **Google account to log into a random app** ("Sign in with Google") — the app never sees or stores your Google password; it just receives a token proving Google verified you, and in AWS's case, exchanges that token for a **temporary AWS access pass** scoped to only what that app is allowed to do.

![[Pasted image 20260805163659.png]]

### Benefits of Identity Federation in AWS (from diagram)

**01 — Simplified User Management**  
Users access AWS resources using their **existing organizational credentials** — no need to create/manage a separate AWS-specific identity for every person.

**02 — Centralized Authentication**  
Administrators manage user access from a **single, central location** (the IdP) — rather than managing permissions separately across every AWS account.

**03 — Improved Security**  
Reduces the risk of errors and improves overall security posture — mainly because federation issues only **temporary credentials** (via STS) instead of long-term access keys that could be leaked, forgotten, or mismanaged.

![[Pasted image 20260805163738.png]]
## Benefits of Identity Federation in AWS (from diagram)

**01 — Simplified User Management** Users access AWS resources using their **existing organizational credentials** — no need to create/manage a separate AWS-specific identity for every person.

**02 — Centralized Authentication** Administrators manage user access from a **single, central location** (the IdP) — rather than managing permissions separately across every AWS account.

**03 — Improved Security** Reduces the risk of errors and improves overall security posture — mainly because federation issues only **temporary credentials** (via STS) instead of long-term access keys that could be leaked, forgotten, or mismanaged.

---

# 📘 Full Summary Notes: AWS Identity Federation

## What It Is

Identity Federation lets users access AWS using an **existing identity** from an external identity provider (IdP) — instead of creating a separate IAM user for every person. Authentication happens with the external IdP; AWS only trusts a token as proof and issues **temporary credentials via STS**.

## Why It's Used (Benefits)

|Benefit|Explanation|
|---|---|
|Simplified User Management|No need to create/manage individual IAM users per person|
|Centralized Authentication|Access managed from one central IdP, not scattered per-account|
|Improved Security|Temporary credentials only — no long-term keys to leak or rotate|

## The Two Types of Federation

||**Workforce Federation**|**Web/Mobile Identity Federation**|
|---|---|---|
|**Who it's for**|Employees|Customers / app users|
|**Protocol**|SAML 2.0|OpenID Connect (OIDC), built on OAuth 2.0|
|**Common IdPs**|Active Directory (AD FS), Okta, Azure AD|Google, Facebook, Amazon, Apple|
|**AWS-managed service**|IAM Identity Center|Amazon Cognito|
|**STS API used**|`AssumeRoleWithSAML`|`AssumeRoleWithWebIdentity`|

## Key Protocols

- **SAML 2.0** — XML-based assertion standard; used for enterprise/workforce SSO.
- **OpenID Connect (OIDC)** — Identity layer built on top of OAuth 2.0; verifies **who** the user is (ID Token/JWT).
- **OAuth 2.0** — Authorization framework only; grants **access to resources**, but doesn't verify identity by itself (that's why OIDC exists).

> **Key distinction:** OAuth 2.0 = authorization ("what can you do?"), OIDC = authentication ("who are you?").

## The General Federation Flow (5 Steps — applies to both SAML & OIDC)

1. **User authenticates** with their external IdP (corporate IdP or social provider) using existing credentials.
2. **IdP returns a token** (SAML assertion or OIDC ID token) confirming identity.
3. **Client calls AWS STS**, presenting that token — `AssumeRoleWithSAML` or `AssumeRoleWithWebIdentity`.
4. **STS validates the token** against the pre-configured trust relationship, then issues **temporary credentials** (Access Key, Secret Key, Session Token).
5. **User/app accesses AWS resources** directly using those temporary credentials, scoped to the assumed role's permissions.

## Critical Underlying Piece: Federated Trust

AWS must be explicitly configured to **trust** a given IdP — this is set in the IAM Role's **trust policy**, specifying the IdP as a trusted `Principal`. Without this pre-configured trust, STS will reject the token exchange.

## Real-World Examples

- **Workforce:** Employees log into Okta → SSO into multiple AWS accounts via IAM Identity Center.
- **Customer-facing:** Mobile app users sign in with Google → app uses Cognito to get temporary AWS credentials for direct S3/DynamoDB access.
- **Modern DevOps use case:** GitHub Actions uses OIDC to assume an AWS role for CI/CD — **no long-term AWS keys stored in GitHub secrets at all**.

## Core Takeaway

Identity Federation is fundamentally just **another way to trigger `sts:AssumeRole`** — using an external, already-trusted identity instead of a native IAM identity — enabling secure, temporary, scalable access to AWS without the overhead and risk of managing long-term credentials for every user.