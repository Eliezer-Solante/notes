Here's the general structure using `aws rds create-db-instance`, with the common specs you'd want to set:

## Basic example (MySQL)

```bash
aws rds create-db-instance \
    --db-instance-identifier mydb-instance \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --engine-version 8.0.35 \
    --master-username admin \
    --master-user-password 'YourSecurePassword123!' \
    --allocated-storage 20 \
    --storage-type gp3 \
    --vpc-security-group-ids sg-0123456789abcdef0 \
    --db-subnet-group-name my-subnet-group \
    --availability-zone us-east-1a \
    --backup-retention-period 7 \
    --port 3306 \
    --publicly-accessible \
    --db-name myappdb \
    --no-multi-az \
    --tags Key=Environment,Value=Production
```

## Key flags explained

|Flag|Purpose|
|---|---|
|`--db-instance-identifier`|Unique name for the instance|
|`--db-instance-class`|Size, e.g. `db.t3.micro`, `db.m5.large`, `db.r6g.xlarge`|
|`--engine`|`mysql`, `postgres`, `mariadb`, `oracle-se2`, `sqlserver-ex`, `aurora-mysql`, `aurora-postgresql`|
|`--engine-version`|Specific version (check available ones — see below)|
|`--allocated-storage`|Storage in GB (min varies by engine)|
|`--storage-type`|`gp2`, `gp3`, `io1`/`io2` (provisioned IOPS), or `standard`|
|`--iops`|Required if using io1/io2, or for gp3 above baseline|
|`--vpc-security-group-ids`|Security group(s) controlling network access|
|`--db-subnet-group-name`|Subnet group defining which VPC subnets it can live in|
|`--multi-az` / `--no-multi-az`|High availability standby in another AZ|
|`--publicly-accessible` / `--no-publicly-accessible`|Whether it gets a public endpoint|
|`--backup-retention-period`|Days to keep automated backups (0 disables)|
|`--storage-encrypted`|Encrypt at rest (recommended)|
|`--kms-key-id`|Custom KMS key, if not using default|

## Check valid engine versions first

```bash
aws rds describe-db-engine-versions \
    --engine mysql \
    --query "DBEngineVersions[].EngineVersion" \
    --output table
```

## Check status after creation

```bash
aws rds describe-db-instances \
    --db-instance-identifier mydb-instance \
    --query "DBInstances[0].DBInstanceStatus"
```

It'll show `creating` → `backing-up` → `available` (usually takes 5–10 minutes).

## A note on the password

Don't hardcode the password in scripts/history if you can avoid it — better to use `--manage-master-user-password` (lets RDS manage it in Secrets Manager) instead of `--master-user-password`, or pull it from a secrets manager/env var.

If you tell me the engine (Postgres, MySQL, Aurora, etc.), expected workload, and whether this is prod or dev, I can give you a more tailored spec (right instance class, storage type, Multi-AZ or not, etc.) rather than generic defaults.