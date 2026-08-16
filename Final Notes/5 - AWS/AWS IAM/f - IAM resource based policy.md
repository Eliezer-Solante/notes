![[Pasted image 20260805130341.png]]
![[Pasted image 20260805130410.png]]

![[Pasted image 20260805130534.png]]
**Line-by-line explanation:**

- **`"Version": "2012-10-17"`** — This is a fixed policy language version required by AWS; it's essentially always this date for current IAM policies.
- **`"Effect": "Deny"`** — This statement explicitly **denies** access (rather than granting it).
- **`"Principal"`** — Since this is a _resource_ policy (attached to the S3 bucket itself, not to a user), it must specify **who** the policy applies to. Here, it's the IAM group `accounting` in account `123456789`.
- **`"Action": "s3:delete"`** — The action being denied. _(Note: the actual AWS action name is `s3:DeleteObject` — `s3:delete` isn't a valid IAM action, so this looks like simplified/example syntax rather than something that would work as-is.)_
- **`"Resource"`** — The specific S3 resources this deny applies to:
    - `arn:aws:s3:::accounting1` → the bucket itself
    - `arn:aws:s3:::accounting1/*` → all objects inside the bucket

**What this policy does overall:**  
It explicitly **blocks the "accounting" IAM group from deleting anything** in the `accounting1` S3 bucket — both the bucket and every object within it.

**Key concept being illustrated:** Resource-based policies (like S3 bucket policies) must include a `Principal` element, because — unlike identity-based policies attached to a user/role — a resource policy needs to specify _who_ it's granting or denying access to, since it isn't inherently tied to any single identity.

**Important AWS rule to remember:** In AWS IAM evaluation logic, an explicit **Deny always overrides any Allow** — so even if the accounting group had `s3:*` permissions elsewhere (e.g., via an IAM policy), this bucket policy would still block them from deleting objects in this specific bucket.


![[Pasted image 20260805131243.png]]
![[Pasted image 20260805131323.png]]



![[Pasted image 20260805131334.png]]

==Sample Situation==
Create an IAM policy named `financial-data-access-policy` that grants the `s3:GetObject` permission for any  
S3 bucket with the prefix `financial-data-` for the `financial-team` group.

You can run the `showcreds` command anytime to retrieve these credentials.

To create an IAM policy, sign in to the AWS Management Console and open the IAM console at [https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/policies](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/policies)

1. Click "Create Policy" button.
    
2. Select "JSON" policy editor.
    
3. In the action, enter `["s3:GetObject"]`
    
4. In the resource, enter `["arn:aws:s3:::financial-data-*", "arn:aws:s3:::financial-data-*/*"]`.
    
    Here, `*` is a wildcard character that matches any string so `financial-data-` will match any S3 bucket with the prefix `financial-data-`.
    
    `/*` is a wildcard character that matches any object in the S3 bucket with the prefix `financial-data-` this is to allow the user to access all objects present inside the S3 bucket.
    
5. The final JSON should look like bellow:
    

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Action": ["s3:GetObject"],
            "Resource": ["arn:aws:s3:::financial-data-*", "arn:aws:s3:::financial-data-*/*"]
        }
    ]
}
```

6. Click "Next".
    
7. Name policy as `financial-data-access-policy`.
    
8. Click "Create Policy"