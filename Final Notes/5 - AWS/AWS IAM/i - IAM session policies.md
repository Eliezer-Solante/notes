![[Pasted image 20260805140422.png]]
![[Pasted image 20260805140506.png]]

![[Pasted image 20260805140539.png]]
![[Pasted image 20260805140616.png]]

**IAM Session Policies** are advanced, inline policies passed **at the moment a role or federated user session is created**, which further **restrict** the permissions of that specific session — they don't grant new permissions, only limit existing ones.

**Core concept:**  
When you assume a role (via `sts:AssumeRole`, `sts:AssumeRoleWithSAML`, `sts:AssumeRoleWithWebIdentity`, or `sts:GetFederationToken`), you can optionally pass a **session policy** as a parameter. The effective permissions for that session become the **intersection** of:

1. The role's identity-based permissions policy, **AND**
2. The role's permissions boundary (if one exists), **AND**
3. The session policy passed at assume-time

Just like permissions boundaries, a session policy **cannot expand** permissions beyond what the role already allows — it can only **narrow** them down for that specific session.

**Why they're used:**  
Session policies let you dynamically scope down access for a **single, temporary use case** without needing to create a new role for every possible restriction. This is especially useful when:

- The same role is reused across many different tasks/users, but each session should only get a subset of that role's full permissions.
- You want to grant broad permissions to a role but limit exactly what a particular application, script, or user can do _in that one session_.

**Example scenario:**

- A role `S3AccessRole` has a permissions policy allowing full access to all S3 buckets (`s3:*` on `*`).
- When a specific user assumes this role, you pass a session policy limiting access to only `arn:aws:s3:::project-bucket`.
- **Effective result:** That user's session can only access `project-bucket`, even though the underlying role technically allows access to all buckets.

**How it's passed (example using AWS CLI):**

bash

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/S3AccessRole \
  --role-session-name my-session \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::project-bucket/*"
    }]
  }'
```

**Key distinctions from other policy types:**
![[Pasted image 20260805140724.png]]

**Simple analogy:**  
If the role is a **master key** that opens many doors, a session policy is like **temporarily filing down that key** so it only opens one specific door — just for this one use, without permanently altering the master key itself.


![[Pasted image 20260805140746.png]]

Check the user can upload a file to the s3 bucket before any configuration of session policy using CLI
![[Pasted image 20260805141017.png]]

Create Policy to allow uploading files to the s3 bucket `company1-hr` named `Session_Policy_Upload_File`
![[Pasted image 20260805141328.png]]

Create a role (JohnUploadRole) for John to access the bucket by assuming John for the role
![[Pasted image 20260805141547.png]]

Check at the command prompt and create session using the role
![[Pasted image 20260805141717.png]]
Exporting the Keys and tokens
![[Pasted image 20260805141838.png]]

Run the upload again and verify if upload works
![[Pasted image 20260805142027.png]]