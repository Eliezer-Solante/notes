![[Pasted image 20260805134506.png]]
![[Pasted image 20260805134708.png]]
![[Pasted image 20260805134722.png]]

**IAM Permissions Boundaries** are an advanced IAM feature that sets the **maximum permissions** an IAM user or role can ever have — acting as a ceiling, not a grant of access itself.

**Key concept:**  
A permissions boundary doesn't grant any permissions on its own. It only **limits** what the identity-based policies attached to that user/role are allowed to do. The _effective permissions_ are the **intersection** of:

1. The permissions boundary, **AND**
2. The identity-based policies (the actual policies attached to the user/role)

So even if a user has an attached policy granting `s3:*` (full S3 access), if their permissions boundary only allows `s3:GetObject`, they can **only** perform `s3:GetObject` — the boundary caps what the attached policy can actually do.

**Why they're used:**  
Permissions boundaries are most commonly used to let **less-privileged administrators safely delegate IAM management** — for example:

- Allowing a team lead to create IAM users/roles for their team
- But preventing them from creating a user with more permissions than the boundary allows (like escalating to admin access)

This prevents **privilege escalation**, since even if a delegated admin _tries_ to attach an overly broad policy, the boundary will cap it.

**Simple analogy:**  
Think of the boundary as a **fence** around a yard (the maximum possible area), and the identity-based policy as the **furniture placement** within it. No matter how the furniture (permissions) is arranged, it can never go outside the fence (boundary).

**Important distinctions:**

||Permissions Boundary|Identity-based Policy|
|---|---|---|
|Grants access?|❌ No — only limits|✅ Yes — grants access|
|Applies to|Users and roles only|Users, roles, and groups|
|Purpose|Sets a maximum limit|Defines actual permissions|

**Quick example:**

- Identity policy: `s3:*` on all resources
- Permissions boundary: `s3:GetObject`, `s3:ListBucket` only
- **Effective access:** Only `GetObject` and `ListBucket` — nothing else, despite the broader identity policy

This is different from a **Deny statement**, which explicitly blocks specific actions — a boundary instead defines the outer limit of what's possible, regardless of what's explicitly allowed elsewhere.

