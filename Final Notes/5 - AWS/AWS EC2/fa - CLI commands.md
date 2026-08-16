to find vpc ID
```bash
aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text
```

to find the number of subnets
```bash
aws ec2 describe-subnets --query 'length(Subnets[])' --region us-east-1
```

to find the id of a selected subnet (in this `us-east-1c` subnet)
```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)" "Name=availability-zone,Values=us-east-1c" --query 'Subnets[].SubnetId' --output text
```

running an ec2 instance with these specifications:
- Use Amazon Linux 2 AMI to get the list of AMIs available using the below command. (You can use any of it)
    ```bash
    aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-2.0.*x86_64-gp2" --query 'Images[*].{ID:ImageId}' --output text
    ``` 
- Instance type should be of `t2.micro`.
- Use subnet of availability zone `us-east-1c`.
- Use the Volume size of `10 Gib` with the volume type of `gp2`.
- Provide the tags of `Name = ec2-kodekloud`.
```bash
aws ec2 run-instances \
    --image-id ami-05f408238af346b4f \
    --instance-type t2.micro \
    --subnet-id subnet-0bf1f7071f1b80272 \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":10,"VolumeType":"gp2"}}]' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-kodekloud}]' \
    --count 1
```

to see the ec2 instance id
```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=ec2-kodekloud" --query 'Reservations[].Instances[].InstanceId' --output text
```

to terminate the ec2 instance 
```
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
```

---

### SAMPLE SITUATIONS

#### **<mark style="background: #FF5582A6;">Create an EC2 instance using AWS CLI under default VPC with the following details:</mark>**

1) The name of the instance must be `nautilus-ec2`.
2) You can use the `ami-0cd59ecaf368e5ccf` AMI to launch this instance.
3) The Instance type must be `t2.micro`.
4) Create a new RSA key pair named `nautilus-kp`.
5) Attach the default (available by default) security group.

##### **EC2 Instance Setup — nautilus-ec2**

**Goal:** Launch a `t2.micro` EC2 instance in the default VPC using a new RSA key pair and the default security group.

1. **Create RSA key pair (`nautilus-kp`)**
```bash
aws ec2 create-key-pair --key-name nautilus-kp --key-type rsa \
  --query 'KeyMaterial' --output text > nautilus-kp.pem
chmod 400 nautilus-kp.pem
```

2. **Get default security group ID**
```bash
aws ec2 describe-security-groups \
  --filters Name=group-name,Values=default \
  --query 'SecurityGroups[0].GroupId' --output text
```

3. **Launch instance**
```bash
aws ec2 run-instances \
  --image-id ami-0cd59ecaf368e5ccf \
  --instance-type t2.micro \
  --key-name nautilus-kp \
  --security-group-ids <sg-id-from-step-2> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]'
```

4. **Verify**
<mark style="background: #BBFABBA6;">Without instance ID</mark>
```bash
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=nautilus-ec2 \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType}' \
  --output table
```

**Key notes:**
- No subnet specified → AWS auto-picks default subnet in default VPC.
- Use `--security-group-ids` (ID-based), not `--security-groups` (name-based), for reliability.
- `.pem` file must be kept secure (chmod 400) — needed for SSH access later.


#### **<mark style="background: #FF5582A6;">EC2 Instance Type Modification — xfusion-ec2</mark>**
    Instruction
1) Change the instance type from `t2.micro` to `t2.nano` for `xfusion-ec2` instance through `aws-cli` only.
2) Make sure the ec2 instance `xfusion-ec2` is in `running` state after the change.

**Goal:** Stop instance `xfusion-ec2` (ID: `i-0dae56330619184e2`) and change its type from `t2.micro` to `t2.nano`, then restart.

**1. Stop the instance**
```bash
aws ec2 stop-instances --instance-ids i-0dae56330619184e2
```

**2. Wait until fully stopped**
```bash
aws ec2 wait instance-stopped --instance-ids i-0dae56330619184e2
```

**3. Modify instance type to t2.nano**
```bash
aws ec2 modify-instance-attribute \
  --instance-id i-0dae56330619184e2 \
  --instance-type "{\"Value\": \"t2.nano\"}"
```

**4. Start the instance again**
```bash
aws ec2 start-instances --instance-ids i-0dae56330619184e2
```

**5. Verify the change**
<mark style="background: #BBFABBA6;">With Instance ID</mark>
```bash
aws ec2 describe-instances \
  --instance-ids i-0dae56330619184e2 \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,State:State.Name,Type:InstanceType}' \
  --output table
```

**Key notes:**

- Instance must be **stopped** before changing type — can't modify while running.
- `t2.nano` has less CPU/memory than `t2.micro` — make sure the workload fits before downsizing.
- Verify step confirms instance name (`xfusion-ec2`), state, and updated type in one table.

#### **<mark style="background: #FF5582A6;">EC2 Instance Deletion — datacenter-ec2 (us-east-1)</mark>**
1) Using AWS CLI, delete an ec2 instance named `datacenter-ec2` present in `us-east-1` region.
2) Before submitting your task, make sure instance is in `terminated` state.

**Goal:** Terminate the EC2 instance named `datacenter-ec2` in the `us-east-1` region, and confirm it reaches `terminated` state before completing the task.

**1. Find the instance ID by name tag**
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters Name=tag:Name,Values=datacenter-ec2 \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text
```

**2. Terminate the instance**
```bash
aws ec2 terminate-instances \
  --region us-east-1 \
  --instance-ids <instance-id-from-step-1>
```

**3. Wait until fully terminated**
```bash
aws ec2 wait instance-terminated \
  --region us-east-1 \
  --instance-ids <instance-id-from-step-1>
```

**4. Verify state = terminated**
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids <instance-id-from-step-1> \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,State:State.Name}' \
  --output table
```

**Key notes:**

- Always pass `--region us-east-1` explicitly (or ensure CLI default region/profile is set to it).
- Task isn't complete until `State.Name` shows `terminated` — the `wait instance-terminated` command blocks until this happens, then the verify step confirms it.
- Termination is irreversible; root EBS volume is deleted by default unless `DeleteOnTermination=false`.