to get the meta-data of `iam/security-credentials` for the EC2 instance you created

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/

```
to list all s3 bucket and filter for the bucket name
```bash
aws s3 ls | grep -oE "kk-.+"
```

to download a file from an s3 bucket from the EC2 instance
```bash
aws s3 cp s3://kk-<random-number>/index.html .

```

to update/upload the modified file to the s3 bucket 
```bash
aws s3 cp index.html s3://kk-<random-number>/index.html

```

to see the kk lab terminal IP run
```bash
curl http://ifconfig.io
```

to run a script for creating an EC2 instance:
```bash
cd /app/terraform_files/stack && terraform init && terraform apply -auto-approve
```
Note: you need to have a script first!!

to install CloudWatch Agent
```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb sudo dpkg -i amazon-cloudwatch-agent.deb
```
to verify it 
```bash
amazon-cloudwatch-agent-ctl -a status
```


Here's the structure using `aws ec2 run-instances`, with common specs:

## Basic example

```bash
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0 \
    --associate-public-ip-address \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":20,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
    --count 1 \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-server},{Key=Environment,Value=Production}]'
```

## Key flags explained

|Flag|Purpose|
|---|---|
|`--image-id`|AMI ID (OS image) — see below for finding one|
|`--instance-type`|Size, e.g. `t3.micro`, `m5.large`, `c6g.xlarge`|
|`--key-name`|SSH key pair name (for Linux) — must already exist|
|`--security-group-ids`|Security group(s) controlling inbound/outbound traffic|
|`--subnet-id`|Which VPC subnet to launch into|
|`--associate-public-ip-address`|Assign a public IP (needs `--subnet-id`, won't work with `--count` >1 the same way — use a network-interfaces spec for that)|
|`--block-device-mappings`|Root/extra EBS volumes, size, type, encryption|
|`--iam-instance-profile`|Attach an IAM role, e.g. `Name=my-ec2-role`|
|`--user-data`|Bootstrap script, e.g. `file://startup.sh`|
|`--count`|How many instances to launch|
|`--tag-specifications`|Tags applied at launch (name, environment, etc.)|
|`--placement`|AZ, e.g. `AvailabilityZone=us-east-1a`|
|`--instance-market-options`|For Spot instances|

## Find a current AMI (e.g. latest Amazon Linux 2023)

```bash
aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
    --query "sort_by(Images, &CreationDate)[-1].ImageId" \
    --output text
```

## With an IAM role and user-data script

```bash
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --instance-type t3.small \
    --key-name my-key-pair \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0 \
    --iam-instance-profile Name=my-ec2-role \
    --user-data file://startup.sh \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":30,"VolumeType":"gp3","Encrypted":true}}]' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server}]'
```

## Check status after launch

```bash
aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=my-server" \
    --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,PublicIP:PublicIpAddress}" \
    --output table
```

A few things worth deciding upfront: OS (Amazon Linux, Ubuntu, etc.), whether it needs a public IP or stays private behind a bastion/NAT, and whether it needs an IAM role for talking to other AWS services (S3, RDS, etc.). Tell me the use case and I can tighten this up.