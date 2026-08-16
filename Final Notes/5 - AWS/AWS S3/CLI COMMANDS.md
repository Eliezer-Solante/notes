# AWS S3 CLI Command Reference

## Bucket Management

```bash
# Create a bucket
aws s3 mb s3://my-bucket-name --region us-west-2

# Remove an empty bucket
aws s3 rb s3://my-bucket-name --region us-west-2

# Remove a bucket and all its contents
aws s3 rb s3://my-bucket-name --force --region us-west-2

# List all buckets (region not needed — account-wide)
aws s3 ls

# List contents of a bucket
aws s3 ls s3://my-bucket-name --region us-west-2

# List contents recursively, with human-readable sizes and summary
aws s3 ls s3://my-bucket-name --recursive --human-readable --summarize --region us-west-2
```

---

## Local Machine → S3 Bucket (Upload)

```bash
# Upload a single file
aws s3 cp ./file.txt s3://my-bucket-name/file.txt --region us-west-2

# Upload to a "folder" (prefix) in the bucket
aws s3 cp ./file.txt s3://my-bucket-name/uploads/file.txt --region us-west-2

# Upload an entire local folder recursively
aws s3 cp ./my-local-folder s3://my-bucket-name/my-folder --recursive --region us-west-2

# Upload with storage class + server-side encryption + ACL
aws s3 cp ./file.txt s3://my-bucket-name/file.txt \
  --region us-west-2 \
  --storage-class STANDARD_IA \
  --sse AES256 \
  --acl private

# Upload only specific file types (exclude all, then include pattern)
aws s3 cp ./my-local-folder s3://my-bucket-name/my-folder \
  --recursive \
  --region us-west-2 \
  --exclude "*" \
  --include "*.jpg"

# Upload and set content-type explicitly
aws s3 cp ./index.html s3://my-bucket-name/index.html \
  --region us-west-2 \
  --content-type "text/html"

# Move local file to bucket (deletes local copy after upload)
aws s3 mv ./file.txt s3://my-bucket-name/file.txt --region us-west-2

# Move an entire local folder to bucket
aws s3 mv ./my-local-folder s3://my-bucket-name/my-folder --recursive --region us-west-2
```

---

## S3 Bucket → Local Machine (Download)

```bash
# Download a single file
aws s3 cp s3://my-bucket-name/file.txt ./file.txt --region us-west-2

# Download to current directory (keep same filename)
aws s3 cp s3://my-bucket-name/file.txt . --region us-west-2

# Download an entire folder/prefix recursively
aws s3 cp s3://my-bucket-name/my-folder ./my-local-folder --recursive --region us-west-2

# Download only specific file types
aws s3 cp s3://my-bucket-name/my-folder ./my-local-folder \
  --recursive \
  --region us-west-2 \
  --exclude "*" \
  --include "*.pdf"

# Download a specific object version
aws s3api get-object \
  --bucket my-bucket-name \
  --key file.txt \
  --version-id VERSION_ID \
  --region us-west-2 \
  ./file.txt

# Move bucket file to local (deletes from bucket after download)
aws s3 mv s3://my-bucket-name/file.txt ./file.txt --region us-west-2
```

---

## Sync (Local ↔ Bucket — only transfers changed/new files)

```bash
# Sync local folder up to bucket
aws s3 sync ./my-local-folder s3://my-bucket-name/my-folder --region us-west-2

# Sync bucket down to local folder
aws s3 sync s3://my-bucket-name/my-folder ./my-local-folder --region us-west-2

# Sync up, deleting files in bucket that no longer exist locally
aws s3 sync ./my-local-folder s3://my-bucket-name/my-folder --delete --region us-west-2

# Sync down, deleting local files no longer in bucket
aws s3 sync s3://my-bucket-name/my-folder ./my-local-folder --delete --region us-west-2

# Preview sync without transferring (dry run)
aws s3 sync ./my-local-folder s3://my-bucket-name/my-folder --dryrun --region us-west-2

# Sync excluding certain files/folders
aws s3 sync ./my-local-folder s3://my-bucket-name/my-folder \
  --region us-west-2 \
  --exclude "*.log" \
  --exclude "node_modules/*"

# Sync comparing by size only (faster, less precise than timestamp check)
aws s3 sync ./my-local-folder s3://my-bucket-name/my-folder \
  --region us-west-2 \
  --size-only
```

---

## Copying / Moving Within S3 (Bucket to Bucket)

```bash
# Copy an object between buckets (different regions)
aws s3 cp s3://source-bucket/file.txt s3://dest-bucket/file.txt \
  --source-region us-east-1 \
  --region us-west-2

# Move an object within the same bucket
aws s3 mv s3://my-bucket-name/file.txt s3://my-bucket-name/archive/file.txt --region us-west-2

# Move/copy an entire folder between buckets
aws s3 mv s3://my-bucket-name/folder s3://my-bucket-name/new-folder \
  --recursive \
  --region us-west-2
```

---

## Deleting Objects

```bash
# Delete a single object
aws s3 rm s3://my-bucket-name/file.txt --region us-west-2

# Delete all objects under a prefix
aws s3 rm s3://my-bucket-name/folder --recursive --region us-west-2

# Dry run delete (preview what would be deleted)
aws s3 rm s3://my-bucket-name/folder --recursive --region us-west-2 --dryrun
```

---

## `s3api` Commands (Low-Level, More Control/Metadata)

```bash
# Create a bucket (region required in LocationConstraint for non us-east-1)
aws s3api create-bucket \
  --bucket my-bucket-name \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2

# Get bucket region/location
aws s3api get-bucket-location --bucket my-bucket-name --region us-west-2

# Bucket policy
aws s3api get-bucket-policy --bucket my-bucket-name --region us-west-2
aws s3api put-bucket-policy --bucket my-bucket-name --region us-west-2 --policy file://policy.json

# Versioning
aws s3api put-bucket-versioning --bucket my-bucket-name --region us-west-2 \
  --versioning-configuration Status=Enabled
aws s3api list-object-versions --bucket my-bucket-name --region us-west-2

# List objects (v2, with prefix filter and max items)
aws s3api list-objects-v2 \
  --bucket my-bucket-name \
  --region us-west-2 \
  --prefix "logs/" \
  --max-items 50

# Get object metadata without downloading
aws s3api head-object --bucket my-bucket-name --key file.txt --region us-west-2

# Get an object (basic retrieval)
aws s3api get-object --bucket my-bucket-name --key file.txt --region us-west-2 output.txt

# Set bucket ACL
aws s3api put-bucket-acl --bucket my-bucket-name --region us-west-2 --acl public-read

# Enable static website hosting
aws s3api put-bucket-website \
  --bucket my-bucket-name \
  --region us-west-2 \
  --website-configuration file://website.json

# Put a bucket lifecycle configuration
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket-name \
  --region us-west-2 \
  --lifecycle-configuration file://lifecycle.json

# Enable server-side encryption (SSE-S3)
aws s3api put-bucket-encryption \
  --bucket my-bucket-name \
  --region us-west-2 \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'

# Generate a presigned URL (valid for 3600 seconds)
aws s3 presign s3://my-bucket-name/file.txt --region us-west-2 --expires-in 3600

# Delete a specific object version
aws s3api delete-object \
  --bucket my-bucket-name \
  --key file.txt \
  --version-id VERSION_ID \
  --region us-west-2

# Delete multiple objects at once
aws s3api delete-objects \
  --bucket my-bucket-name \
  --region us-west-2 \
  --delete file://delete-list.json
```

---

## Quick Tips

- Use `aws s3` (`cp`, `sync`, `mv`, `rm`) for everyday file transfers — simplest syntax.
- Use `aws s3api` when you need fine-grained control (metadata, versioning, encryption, lifecycle rules).
- `--dryrun` works with `cp`, `sync`, `mv`, and `rm` — always good to preview before bulk operations.
- `--region` matters most for `create-bucket`; other commands on existing buckets usually auto-resolve the correct region.
- `--recursive` is required on `cp`, `mv`, and `rm` when working with folders/prefixes (not needed for `sync`, which is always recursive).
- `sync` only transfers changed or new files — much more efficient than `cp --recursive` for repeated transfers.


---
At bucket creation, you can only set the **overall public access posture** — not per-object public/private, since that requires policies (which you said to skip). Here's what you can control right at creation time:

## Create a bucket with public access blocked (fully private — default & recommended)

```bash
aws s3api create-bucket \
  --bucket my-bucket-name \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2

aws s3api put-public-access-block \
  --bucket my-bucket-name \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

## Create a bucket allowing public access (open for public/private mix later)

```bash
aws s3api create-bucket \
  --bucket my-bucket-name \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2 \
  --object-ownership BucketOwnerPreferred

aws s3api put-public-access-block \
  --bucket my-bucket-name \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

This just **unlocks the ability** for the bucket to have public and private content mixed — it doesn't make anything public by itself. You'd still need per-object ACLs or a policy afterward to actually expose specific files/folders. But since you said to skip policies, here's the ACL-based way instead:

## Set individual objects public/private via ACL (no policy needed)

```bash
# Upload a private object (default)
aws s3 cp ./private-file.txt s3://my-bucket-name/private-file.txt --acl private

# Upload a public object
aws s3 cp ./public-file.txt s3://my-bucket-name/public-file.txt --acl public-read
```

**Note:** for `--acl public-read` to actually work, the bucket must have `BlockPublicAcls=false` and `IgnorePublicAcls=false` set (from the second creation example above). Also, ACLs only work if **Object Ownership** isn't set to `BucketOwnerEnforced` (the modern default for new buckets, which disables ACLs entirely) — that's why the example explicitly sets `--object-ownership BucketOwnerPreferred`.

## Quick check after creation

```bash
aws s3api get-public-access-block --bucket my-bucket-name
aws s3api get-bucket-ownership-controls --bucket my-bucket-name
```