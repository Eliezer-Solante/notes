# AWS Core Services Fundamentals — Combined Solution

## Part 1 — Build the Dev Environment

**1.1–1.3 VPC & Subnets** Console: VPC → Your VPCs → Create VPC → "VPC only" → Name `dev`, IPv4 CIDR `10.0.0.0/23`, no IPv6 → tags `environment=dev`, `team=silver` → Create. VPC → Subnets → Create subnet → VPC `dev` → name `web-a`, AZ `us-east-1a`, CIDR `10.0.0.0/24`, same tags. Add another: name `web-b`, AZ `us-east-1b`, CIDR `10.0.1.0/24`, same tags.

**2.1 Internet Gateway** VPC → Internet Gateways → Create → name `dev-internet-gateway`, tags → Create. Select it → Actions → Attach to VPC → `dev`.

**3.1–3.2 Route Table** VPC → Route Tables → Create → name `internet-route`, VPC `dev`, tags → Create. Routes tab → Edit routes → Add `0.0.0.0/0` → Target: Internet Gateway → `dev-internet-gateway` → Save. Subnet Associations tab → Edit → check `web-a` and `web-b` → Save. Subnets → select `web-a` → Actions → Edit subnet settings → enable auto-assign public IPv4 → Save. Repeat for `web-b`.

**4.1–4.2 Security Groups** EC2 → Security Groups → Create → name `allow-http`, description "Allows HTTP traffic", VPC `dev`, inbound rule HTTP from 0.0.0.0/0, tags → Create. Create another: name `allow-https`, description "Allows HTTPS traffic", inbound rule HTTPS from 0.0.0.0/0, tags → Create.

**5.1–5.4 S3 Buckets**

```
date
```

Note `MMDDYYYY`. Console: S3 → Create bucket → `kodekloud-ow-pave-[username]-[MMDDYYYY]-b1`, region us-east-1 → Create → Properties → Tags → add `environment=dev`, `team=silver`. Upload `picture.png` (any PNG <5MB) via the bucket's Upload button. Create second bucket `...b2`, same tags. Open it → Properties → Bucket Versioning → Edit → Enable.

**6.1–6.2 RDS Instances** RDS → Create database → Standard create → MySQL 8.0.45, Free tier, ID `dev-mysql-db`, user `pave_admin`, password `#An2qG8vCH2QGVrWffu3Cb&$`, `db.t3.micro`, gp2 20GiB, VPC `dev`, Public access No, SG `default`, tags → Create. Repeat: PostgreSQL 16.13-R1, ID `dev-postgres-db`, user `pave_admin`, password `LQ6*kCCCG4gbUBECc$54MKYSZ`, same class/storage/VPC/tags → Create.

**7.1–7.2 EC2 Instances** EC2 → Launch instances → name `Web-1`, Amazon Linux AMI, `t2.micro`, no key pair, subnet `web-a`, existing SG `allow-http`, tags → Launch. Launch another: name `Web-2`, Ubuntu AMI, `t2.medium`, no key pair, subnet `web-b`, existing SG `allow-https`, tags → Launch.

**8.1–8.2 CloudWatch/SNS** SNS → Topics → Create → Standard, name `silver-alerts`, tags → Create. CloudWatch → Alarms → Create alarm → EC2 Per-Instance Metrics → `Web-1` → CPUUtilization → condition ≥80 → notification: `silver-alerts` → name `web-1-cpu-alarm` → Create. Add tags if the review page offers it, else:

```
aws cloudwatch tag-resource --resource-arn <alarm-arn> --tags Key=environment,Value=dev Key=team,Value=silver
```

---

## Part 2 — Troubleshoot & Restore HR Portal

**9.1 Deploy environment**

```
curl -O https://aws-assessment-resources.ap-south-1.linodeobjects.com/setup.tar.gz
tar -xzvf setup.tar.gz
chmod +x setup
./setup
```

(In your case the working file ended up being served as `setup-test` — same idea, just chmod + run it.)

Deployment printed:

```
EC2 Instance Name : HR-Portal-App
Security Group    : hr-portal-sg-14499
IAM Role          : hr-portal-s3-role-14499
S3 Bucket         : hr-portal-files-14499
Application URL   : http://98.93.127.134
```

**9.2 Investigate and fix**

Diagnosed two separate misconfigurations:

1. **Security group missing HTTP rule** — only port 22 (SSH) was open, so the app was completely unreachable over the web.

```
aws ec2 authorize-security-group-ingress --group-id sg-08e7482943b766045 --protocol tcp --port 80 --cidr 0.0.0.0/0
```

2. **IAM instance profile never attached to the EC2 instance** — the role (`hr-portal-s3-role-14499`) and its policy (correct: `s3:GetObject`, `s3:ListBucket` on `hr-portal-files-14499`) both existed and were valid, but `describe-iam-instance-profile-associations` came back empty, meaning the instance had no credentials to actually use them.

```
aws ec2 associate-iam-instance-profile --instance-id i-06fdf33758880589a --iam-instance-profile Name=hr-portal-profile-14499
```

**Result:** `http://98.93.127.134` now shows **[PASS] HR Portal Restored!** — confirmed the app successfully retrieves its data from S3.

---

## Final one-shot submission (do only when fully confident)

```
curl -O https://aws-assessment-resources.ap-south-1.linodeobjects.com/aws-checker
chmod +x aws-checker
ls -l aws-checker    # sanity-check file size before running
./aws-checker
```


==Detailed commands for part 2==

## Part 2 — All Commands Used, In Order


**Deploy**

```
curl -O https://aws-assessment-resources.ap-south-1.linodeobjects.com/setup-test
chmod +x setup-test
./setup-test
```

**Investigate the EC2 instance**

```
aws ec2 describe-instances --filters "Name=tag:Name,Values=HR-Portal-App" --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress,SecurityGroups]'
```

**Investigate the security group**

```
aws ec2 describe-security-groups --group-names hr-portal-sg-14499
```

**Fix 1 — open HTTP (port 80)**

```
aws ec2 authorize-security-group-ingress --group-id sg-08e7482943b766045 --protocol tcp --port 80 --cidr 0.0.0.0/0
```

**Investigate the IAM role**

```
aws iam get-role --role-name hr-portal-s3-role-14499
aws iam list-role-policies --role-name hr-portal-s3-role-14499
aws iam list-attached-role-policies --role-name hr-portal-s3-role-14499
```

**Investigate the IAM policy content**

```
aws iam get-policy --policy-arn arn:aws:iam::891377344088:policy/hr-portal-s3-policy-14499
aws iam get-policy-version --policy-arn arn:aws:iam::891377344088:policy/hr-portal-s3-policy-14499 --version-id v1
```

**Check if the instance profile was attached (found the root cause here — empty result)**

```
aws ec2 describe-iam-instance-profile-associations --filters "Name=instance-id,Values=i-06fdf33758880589a"
```

**Find the instance profile name**

```
aws iam list-instance-profiles-for-role --role-name hr-portal-s3-role-14499
```

**Fix 2 — attach the instance profile to the instance (this resolved it)**

```
aws ec2 associate-iam-instance-profile --instance-id i-06fdf33758880589a --iam-instance-profile Name=hr-portal-profile-14499
```

**Final one-shot submission (once fully verified — not yet run)**

```
curl -O https://aws-assessment-resources.ap-south-1.linodeobjects.com/aws-checker
chmod +x aws-checker
ls -l aws-checker
./aws-checker
```