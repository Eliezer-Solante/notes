That's exactly it — SSH is refusing to load your private key because its file permissions are too permissive. Fix it with:

```
chmod 600 ~/.ssh/id_ed25519
```

SSH from your IP only 
`curl -s ifconfig.me`



`scp -i mykeypair.pem ~/id_ed25519 ~/id_ed25519.pub ec2-user@34.203.212.161:~/` 
This lands both files in the ec2-user's home directory first — scp can't write directly into a remote `.ssh` folder with correct ownership/permissions in one step reliably, so staging in home first is safest.


### GROUP ACTIVITY
**Activity 1 — Prepare the AWS Environment**

Console:

- IAM → Users → your user → Security credentials tab → Create access key → choose "Command Line Interface (CLI)" → save both values.
- EC2 → Key Pairs → Create key pair → name `employee-app-key`, type RSA, format `.pem`, add tag Key `Project` Value `employee-app` → Create (downloads the `.pem` automatically).
- EC2 → Security Groups → Create security group → name `employee-app-sg`, description "SG for Employee Records Portal", VPC: default. Inbound rules: SSH from "My IP", HTTP from Anywhere-IPv4. Tag Key `Project` Value `employee-app` → Create.

Commands (one-time, in CloudShell or local terminal):

```
aws configure
```

Enter Access Key ID, Secret Access Key, region `us-east-1`, output `json`.

Evidence:

```
aws sts get-caller-identity > activity1-identity.txt
aws ec2 describe-key-pairs > activity1-keypair.txt
aws ec2 describe-security-groups > activity1-securitygroups.txt
```

---

**Activity 2 — Migrate the Application to EC2**

Console:

- EC2 → Instances → Launch instances → name `employee-app-server`, AMI: Amazon Linux 2023, type `t2.micro`, key pair `employee-app-key`, security group: existing `employee-app-sg`, tag Key `Project` Value `employee-app` → Launch instance.
- Wait for Instance state = Running, Status checks = 2/2 passed. Copy the Public IPv4 address.

Commands:

```
ssh -i employee-app-key.pem ec2-user@<PUBLIC_IP>
```

On the instance:

```
sudo dnf update -y
sudo dnf install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
```

Get the repo onto the instance:

```
sudo dnf install git -y
git clone https://github.com/opswerks-academy/app-db-aws-test.git
```

Deploy and verify:

```
sudo cp application/index.html /var/www/html/index.html
curl http://localhost
```

Evidence:

```
aws ec2 describe-instances > activity2-ec2.txt
```

Plus a browser screenshot of `http://<PUBLIC_IP>`.

---

**Activity 3 — Migrate the Database to RDS**

Console:

- RDS → Subnet groups → Create DB subnet group → name `employee-app-db-subnets`, VPC: default, select at least 2 AZs and their subnets → Create.
- EC2 → Security Groups → Create security group → name `employee-app-db-sg`, VPC: default. Inbound rule: MySQL/Aurora (3306), source: custom, select `employee-app-sg`. Tag Key `Project` Value `employee-app` → Create.
- RDS → Databases → Create database → Standard create, Engine MySQL, template Free tier, identifier `hr-employee-db`, username `admin`, set a password, class `db.t3.micro`. Connectivity: VPC default, DB subnet group `employee-app-db-subnets`, Public access: No, VPC security group: existing → `employee-app-db-sg`. Tags: Key `Project` Value `employee-app` → Create database. Wait for status = Available.
- Click the instance → Connectivity & security tab → copy the Endpoint.

Commands (on the EC2 instance, inside `database/` folder):

```
sudo dnf install mariadb105 -y
```

```
mysql -h <RDS-ENDPOINT> -P 3306 -u admin -p < employee_db.sql
```

```
mysql -h <RDS-ENDPOINT> -P 3306 -u admin -p employee_db < employee-data.sql
```

```
mysql -h <RDS-ENDPOINT> -P 3306 -u admin -p
```

Then:

```
USE employee_db;
SELECT * FROM employees;
EXIT;
```

```
nano db.conf
```

Set `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS`, save (`Ctrl+O`, Enter, `Ctrl+X`).

```
chmod 600 db.conf
chmod +x add-employee.sh update-employee.sh
sudo ./update-employee.sh
```

Evidence:

```
aws rds describe-db-instances > activity3-rds.txt
aws ec2 describe-security-groups > activity3-securitygroups.txt
```

Plus a browser screenshot of `http://<PUBLIC_IP>` showing RDS-sourced data.

---

**Activity 4 — Additional Storage using EBS**

Console:

- EC2 → Instances → select `employee-app-server` → note the Availability Zone.
- EC2 → Volumes → Create volume → type gp3, size 1 GiB, matching AZ, tag Key `Project` Value `employee-app` → Create volume.
- Volumes → select the new volume → Actions → Attach volume → select `employee-app-server`, leave default device name → Attach.

Commands (on the instance):

```
lsblk
```

Identify the device (e.g. `/dev/xvdf`).

```
sudo mkfs -t ext4 /dev/xvdf
sudo mkdir /logs
sudo mount /dev/xvdf /logs
df -h /logs
```

Evidence:

```
findmnt /logs > activity4-mount.txt
```

(from CloudShell)

```
aws ec2 describe-volumes > activity4-volumes.txt
```

---

**Activity 5 — Backup Storage using S3**

Console:

- S3 → Create bucket → globally unique name (e.g. `employee-db-backups-<yourname>-2026`), region us-east-1 → Create bucket. Then Properties tab → Tags → Edit → add Key `Project` Value `employee-app`.
- IAM → Roles → Create role → trusted entity AWS service, use case EC2 → attach `AmazonS3FullAccess` (or a scoped custom policy) → name `employee-app-ec2-s3-role`, tag Key `Project` Value `employee-app` → Create role.
- EC2 → Instances → select `employee-app-server` → Actions → Security → Modify IAM role → select `employee-app-ec2-s3-role` → Update IAM role.

Commands (on the instance, inside `backup/` folder):

```
nano db.conf
```

Set RDS connection details, save.

```
chmod 600 db.conf
chmod +x backup.sh
./backup.sh <your-bucket-name>
```

Evidence:

```
aws s3 ls s3://<your-bucket-name>/
```

(from CloudShell)

```
aws s3 ls > activity5-s3.txt
```

Plus a screenshot of the bucket showing the uploaded `.sql` file.

---

**Activity 6 — CloudWatch Alarm**

Console:

- CloudWatch → Metrics → All metrics → EC2 → Per-Instance Metrics → find your instance → select CPUUtilization.
- From the graph: Actions → Create alarm (or Alarms → All alarms → Create alarm). Condition: Static, Greater than 80. Datapoints to alarm: 2 out of 2. Next.
- Notification step: remove any auto-added SNS topic (not required) → Next.
- Name: `employee-app-high-cpu`, description "Alert when CPU exceeds 80%" → Next → Create alarm.
- If a Tags section appears on the review page, add Key `Project` Value `employee-app`.

Commands: none required — this activity is fully console-driven.

Evidence (from CloudShell):

```
aws cloudwatch describe-alarms > activity6-cloudwatch.txt
```