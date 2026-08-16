
## 1. IAM Users

```bash
# Create/Delete
aws iam create-user --user-name USER_NAME
aws iam delete-user --user-name USER_NAME

# Get info / List
aws iam get-user --user-name USER_NAME
aws iam list-users

# Update
aws iam update-user --user-name OLD_NAME --new-user-name NEW_NAME

# Login profile (console password)
aws iam create-login-profile --user-name USER_NAME --password PASSWORD --password-reset-required
aws iam update-login-profile --user-name USER_NAME --password NEW_PASSWORD
aws iam delete-login-profile --user-name USER_NAME
aws iam get-login-profile --user-name USER_NAME

# Access keys
aws iam create-access-key --user-name USER_NAME
aws iam list-access-keys --user-name USER_NAME
aws iam update-access-key --user-name USER_NAME --access-key-id KEY_ID --status Inactive
aws iam delete-access-key --user-name USER_NAME --access-key-id KEY_ID

# MFA
aws iam create-virtual-mfa-device --virtual-mfa-device-name MFA_NAME
aws iam enable-mfa-device --user-name USER_NAME --serial-number ARN --authentication-code1 CODE1 --authentication-code2 CODE2
aws iam deactivate-mfa-device --user-name USER_NAME --serial-number ARN
aws iam list-mfa-devices --user-name USER_NAME

# Tags
aws iam tag-user --user-name USER_NAME --tags Key=Team,Value=Engineering
aws iam untag-user --user-name USER_NAME --tag-keys Team
```

## 2. IAM User Groups

```bash
# Create/Delete
aws iam create-group --group-name GROUP_NAME
aws iam delete-group --group-name GROUP_NAME

# Get info / List
aws iam get-group --group-name GROUP_NAME
aws iam list-groups

# Manage membership
aws iam add-user-to-group --user-name USER_NAME --group-name GROUP_NAME
aws iam remove-user-from-group --user-name USER_NAME --group-name GROUP_NAME
aws iam list-groups-for-user --user-name USER_NAME

# Attach/Detach managed policies
aws iam attach-group-policy --group-name GROUP_NAME --policy-arn POLICY_ARN
aws iam detach-group-policy --group-name GROUP_NAME --policy-arn POLICY_ARN
aws iam list-attached-group-policies --group-name GROUP_NAME

# Inline policies
aws iam put-group-policy --group-name GROUP_NAME --policy-name POLICY_NAME --policy-document file://policy.json
aws iam get-group-policy --group-name GROUP_NAME --policy-name POLICY_NAME
aws iam delete-group-policy --group-name GROUP_NAME --policy-name POLICY_NAME
aws iam list-group-policies --group-name GROUP_NAME
```

## 3. IAM Policies (Managed)

```bash
# Create/Delete
aws iam create-policy --policy-name POLICY_NAME --policy-document file://policy.json
aws iam delete-policy --policy-arn POLICY_ARN

# Get info / List
aws iam get-policy --policy-arn POLICY_ARN
aws iam list-policies --scope Local          # Customer managed only
aws iam list-policies --scope AWS            # AWS managed only

# Versions (managed policies support up to 5 versions)
aws iam create-policy-version --policy-arn POLICY_ARN --policy-document file://policy-v2.json --set-as-default
aws iam list-policy-versions --policy-arn POLICY_ARN
aws iam get-policy-version --policy-arn POLICY_ARN --version-id v2
aws iam set-default-policy-version --policy-arn POLICY_ARN --version-id v2
aws iam delete-policy-version --policy-arn POLICY_ARN --version-id v1

# See what's attached to a policy
aws iam list-entities-for-policy --policy-arn POLICY_ARN
```

## 4. Inline Policies (User/Role-specific)

```bash
# User inline policies
aws iam put-user-policy --user-name USER_NAME --policy-name POLICY_NAME --policy-document file://policy.json
aws iam get-user-policy --user-name USER_NAME --policy-name POLICY_NAME
aws iam delete-user-policy --user-name USER_NAME --policy-name POLICY_NAME
aws iam list-user-policies --user-name USER_NAME

# Role inline policies
aws iam put-role-policy --role-name ROLE_NAME --policy-name POLICY_NAME --policy-document file://policy.json
aws iam get-role-policy --role-name ROLE_NAME --policy-name POLICY_NAME
aws iam delete-role-policy --role-name ROLE_NAME --policy-name POLICY_NAME
aws iam list-role-policies --role-name ROLE_NAME
```

## 5. Permissions (Attach/Detach Managed Policies to Identities)

```bash
# To a user
aws iam attach-user-policy --user-name USER_NAME --policy-arn POLICY_ARN
aws iam detach-user-policy --user-name USER_NAME --policy-arn POLICY_ARN
aws iam list-attached-user-policies --user-name USER_NAME

# To a role
aws iam attach-role-policy --role-name ROLE_NAME --policy-arn POLICY_ARN
aws iam detach-role-policy --role-name ROLE_NAME --policy-arn POLICY_ARN
aws iam list-attached-role-policies --role-name ROLE_NAME

# List ALL policies (inline + managed) affecting a user
aws iam list-policies-granting-service-access --arn USER_ARN --service-namespaces s3

# Simulate/test permissions before applying
aws iam simulate-principal-policy --policy-source-arn USER_ARN --action-names s3:GetObject --resource-arns arn:aws:s3:::bucket/*
```

## 6. Permission Boundaries

```bash
# Set/Remove on a user
aws iam put-user-permissions-boundary --user-name USER_NAME --permissions-boundary POLICY_ARN
aws iam delete-user-permissions-boundary --user-name USER_NAME

# Set/Remove on a role
aws iam put-role-permissions-boundary --role-name ROLE_NAME --permissions-boundary POLICY_ARN
aws iam delete-role-permissions-boundary --role-name ROLE_NAME

# Check current boundary
aws iam get-user --user-name USER_NAME       # shows PermissionsBoundary field
aws iam get-role --role-name ROLE_NAME        # shows PermissionsBoundary field
```

## 7. IAM Roles

```bash
# Create/Delete
aws iam create-role --role-name ROLE_NAME --assume-role-policy-document file://trust-policy.json
aws iam delete-role --role-name ROLE_NAME

# Get info / List
aws iam get-role --role-name ROLE_NAME
aws iam list-roles

# Update trust policy
aws iam update-assume-role-policy --role-name ROLE_NAME --policy-document file://trust-policy.json

# Instance profiles (needed for EC2 to use a role)
aws iam create-instance-profile --instance-profile-name PROFILE_NAME
aws iam add-role-to-instance-profile --instance-profile-name PROFILE_NAME --role-name ROLE_NAME
aws iam remove-role-from-instance-profile --instance-profile-name PROFILE_NAME --role-name ROLE_NAME
aws iam delete-instance-profile --instance-profile-name PROFILE_NAME
aws iam list-instance-profiles-for-role --role-name ROLE_NAME

# Tags
aws iam tag-role --role-name ROLE_NAME --tags Key=Env,Value=Prod
```

## 8. Sessions (STS — Assuming Roles / Temporary Credentials)

```bash
# Assume a role (basic)
aws sts assume-role --role-arn ROLE_ARN --role-session-name SESSION_NAME

# Assume a role with a session policy (further restricts permissions)
aws sts assume-role --role-arn ROLE_ARN --role-session-name SESSION_NAME \
  --policy file://session-policy.json

# Assume role with MFA
aws sts assume-role --role-arn ROLE_ARN --role-session-name SESSION_NAME \
  --serial-number MFA_ARN --token-code MFA_CODE

# Assume role via SAML federation
aws sts assume-role-with-saml --role-arn ROLE_ARN --principal-arn IDP_ARN \
  --saml-assertion SAML_ASSERTION

# Assume role via web identity (e.g., Cognito, OIDC)
aws sts assume-role-with-web-identity --role-arn ROLE_ARN --role-session-name SESSION_NAME \
  --web-identity-token TOKEN

# Get temporary credentials for federated (IAM) users
aws sts get-federation-token --name USER_NAME --policy file://policy.json

# Get info about current identity/session
aws sts get-caller-identity

# Get a session token (adds MFA to existing long-term creds)
aws sts get-session-token --serial-number MFA_ARN --token-code MFA_CODE
```

**After `assume-role`, export the returned temp credentials:**

```bash
export AWS_ACCESS_KEY_ID=xxx
export AWS_SECRET_ACCESS_KEY=xxx
export AWS_SESSION_TOKEN=xxx
```

## Quick Reference — General Patterns

|Action|Pattern|
|---|---|
|Create|`create-*`|
|Delete|`delete-*`|
|Get single item|`get-*`|
|List multiple items|`list-*`|
|Attach managed policy|`attach-*-policy`|
|Detach managed policy|`detach-*-policy`|
|Inline policy|`put-*-policy` / `get-*-policy` / `delete-*-policy`|
