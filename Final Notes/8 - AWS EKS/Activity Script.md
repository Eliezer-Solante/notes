
```bash
./eliezer-eks-cluster-upgrade-part1.sh identity   2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh vpc        2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh cluster    2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh addons     2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh nodegroup  2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh access     2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh verify     2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
```

### Expanded literal commands with a one-line description each

#### Identity checks

- **Get caller identity**
    

```bash
aws sts get-caller-identity
```

**Description:** Returns the AWS account, user/role ARN, and user ID for the current credentials.

- **Show AWS CLI version**
```
aws --version
```

**Description:** Prints the installed AWS CLI version and runtime info.

- **Show eksctl version**
```
eksctl version
```

**Description:** Prints the installed `eksctl` client version.

- **List supported EKS cluster versions (region us-east-1)**
```
aws eks describe-cluster-versions --region us-east-1 --query "clusterVersions[?status=='STANDARD_SUPPORT'].{Version:clusterVersion,Status:status}" --output table
```
**Description:** Lists EKS Kubernetes versions in standard support for the specified region.

#### VPC and networking (hardcoded example resource IDs)

- **Create VPC**
```
aws ec2 create-vpc --cidr-block 10.10.0.0/24 --region us-east-1 --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=academy-eliezer-vpc},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]"
```
**Description:** Creates a VPC with the specified CIDR and tags.

- **Find VPC ID by tag**
```
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=academy-eliezer-vpc" --query "Vpcs[0].VpcId" --output text --region us-east-1
```
**Description:** Queries the VPC ID that was tagged `academy-eliezer-vpc`.

- **Enable DNS hostnames on VPC**
```
aws ec2 modify-vpc-attribute --vpc-id vpc-0a1b2c3d4e5f6g7h --enable-dns-hostnames '{"Value":true}' --region us-east-1
```
**Description:** Enables DNS hostnames for the VPC (required for EKS).

- **Enable DNS support on VPC**
```
aws ec2 modify-vpc-attribute --vpc-id vpc-0a1b2c3d4e5f6g7h --enable-dns-support '{"Value":true}' --region us-east-1
```
**Description:** Enables DNS resolution for the VPC.

- **Create public subnet us-east-1a**
```
aws ec2 create-subnet --vpc-id vpc-0a1b2c3d4e5f6g7h --cidr-block 10.10.0.0/28 --availability-zone us-east-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=academy-eliezer-public-1a},{Key=kubernetes.io/role/elb,Value=1},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```
**Description:** Creates the first public subnet and tags it for ELB usage.

- **Get subnet ID for academy-eliezer-public-1a**
```
aws ec2 describe-subnets --filters "Name=tag:Name,Values=academy-eliezer-public-1a" --query "Subnets[0].SubnetId" --output text --region us-east-1
```
**Description:** Retrieves the subnet ID by its Name tag.

- **Create public subnet us-east-1b**
```
aws ec2 create-subnet --vpc-id vpc-0a1b2c3d4e5f6g7h --cidr-block 10.10.0.16/28 --availability-zone us-east-1b --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=academy-eliezer-public-1b},{Key=kubernetes.io/role/elb,Value=1},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```
**Description:** Creates the second public subnet.

- **Get subnet ID for academy-eliezer-public-1b**
```
aws ec2 describe-subnets --filters "Name=tag:Name,Values=academy-eliezer-public-1b" --query "Subnets[0].SubnetId" --output text --region us-east-1
```

**Description:** Retrieves the second public subnet ID.

- **Create private subnet us-east-1a**
```
aws ec2 create-subnet --vpc-id vpc-0a1b2c3d4e5f6g7h --cidr-block 10.10.0.64/26 --availability-zone us-east-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=academy-eliezer-private-1a},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```
**Description:** Creates a private subnet for worker nodes and internal ELBs.

- **Get subnet ID for academy-eliezer-private-1a**
```
aws ec2 describe-subnets --filters "Name=tag:Name,Values=academy-eliezer-private-1a" --query "Subnets[0].SubnetId" --output text --region us-east-1
```
**Description:** Retrieves the private subnet ID (1a).

- **Create private subnet us-east-1b**
```
aws ec2 create-subnet --vpc-id vpc-0a1b2c3d4e5f6g7h --cidr-block 10.10.0.128/26 --availability-zone us-east-1b --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=academy-eliezer-private-1b},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```
**Description:** Creates the second private subnet.

- **Get subnet ID for academy-eliezer-private-1b**
```
aws ec2 describe-subnets --filters "Name=tag:Name,Values=academy-eliezer-private-1b" --query "Subnets[0].SubnetId" --output text --region us-east-1
```
**Description:** Retrieves the private subnet ID (1b).

- **Enable auto-assign public IP on public subnet 1a**
```
aws ec2 modify-subnet-attribute --subnet-id subnet-0a1b2c3d4e5f6g7h --map-public-ip-on-launch --region us-east-1
```

**Description:** Configures the subnet to auto-assign public IPs to launched instances.

- **Enable auto-assign public IP on public subnet 1b**
```
aws ec2 modify-subnet-attribute --subnet-id subnet-1b2c3d4e5f6g7h8i --map-public-ip-on-launch --region us-east-1
```
**Description:** Same as above for the second public subnet.

- **Create Internet Gateway**
```
aws ec2 create-internet-gateway --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=academy-eliezer-igw},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```

**Description:** Creates an IGW to provide internet access for public subnets.

- **Get Internet Gateway ID**
```
aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=academy-eliezer-igw" --query "InternetGateways[0].InternetGatewayId" --output text --region us-east-1
```
**Description:** Retrieves the IGW ID by tag.

- **Attach Internet Gateway to VPC**
```
aws ec2 attach-internet-gateway --internet-gateway-id igw-0a1b2c3d4e5f6g7h --vpc-id vpc-0a1b2c3d4e5f6g7h --region us-east-1
```
**Description:** Attaches the IGW to the VPC so public subnets can reach the internet.

- **Allocate Elastic IP for NAT**
```
aws ec2 allocate-address --domain vpc --region us-east-1 --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]"
```
**Description:** Allocates an EIP to be used by the NAT gateway.

- **Get EIP allocation ID by tag**
```
aws ec2 describe-addresses --filters "Name=tag:student,Values=eliezer" --query "Addresses[0].AllocationId" --output text --region us-east-1
```
**Description:** Retrieves the allocation ID of the EIP tagged for this student.

- **Create NAT Gateway in public subnet 1a**
```
aws ec2 create-nat-gateway --subnet-id subnet-0a1b2c3d4e5f6g7h --allocation-id eipalloc-0a1b2c3d4e5f6g7h --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=academy-eliezer-nat},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```
**Description:** Creates a NAT gateway in the public subnet to provide internet access for private subnets.

- **Get NAT Gateway ID by tag**
```
aws ec2 describe-nat-gateways --filter "Name=tag:Name,Values=academy-eliezer-nat" --query "NatGateways[0].NatGatewayId" --output text --region us-east-1
```
**Description:** Retrieves the NAT gateway ID.

- **Wait for NAT gateway to become available**
```
aws ec2 wait nat-gateway-available --nat-gateway-ids "nat-0a1b2c3d4e5f6g7h" --region us-east-1
```
**Description:** Blocks until the NAT gateway is in the `available` state.

- **Create public route table**
```
aws ec2 create-route-table --vpc-id vpc-0a1b2c3d4e5f6g7h --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=academy-eliezer-public-rt},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```
**Description:** Creates a route table for public subnets.

- **Get public route table ID**
```
aws ec2 describe-route-tables --filters "Name=tag:Name,Values=academy-eliezer-public-rt" --query "RouteTables[0].RouteTableId" --output text --region us-east-1
```
**Description:** Retrieves the public route table ID.

- **Create default route to IGW in public route table**
```
aws ec2 create-route --route-table-id rtb-0a1b2c3d4e5f6g7h --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0a1b2c3d4e5f6g7h --region us-east-1
```
**Description:** Adds a default route to the IGW so public subnets have internet access.

- **Associate public route table with public subnet 1a**
```
aws ec2 associate-route-table --route-table-id rtb-0a1b2c3d4e5f6g7h --subnet-id subnet-0a1b2c3d4e5f6g7h --region us-east-1
```
**Description:** Associates the public route table with the first public subnet.

- **Associate public route table with public subnet 1b**
```
aws ec2 associate-route-table --route-table-id rtb-0a1b2c3d4e5f6g7h --subnet-id subnet-1b2c3d4e5f6g7h8i --region us-east-1
```
**Description:** Associates the public route table with the second public subnet.

- **Create private route table**
```
aws ec2 create-route-table --vpc-id vpc-0a1b2c3d4e5f6g7h --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=academy-eliezer-private-rt},{Key=env,Value=training},{Key=team,Value=academy-sre},{Key=student,Value=eliezer}]" --region us-east-1
```
**Description:** Creates a route table for private subnets.

- **Get private route table ID**
```
aws ec2 describe-route-tables --filters "Name=tag:Name,Values=academy-eliezer-private-rt" --query "RouteTables[0].RouteTableId" --output text --region us-east-1
```
**Description:** Retrieves the private route table ID.

- **Create default route to NAT in private route table**
```
aws ec2 create-route --route-table-id rtb-1b2c3d4e5f6g7h8i --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0a1b2c3d4e5f6g7h --region us-east-1
```
**Description:** Routes outbound internet traffic from private subnets through the NAT gateway.

- **Associate private route table with private subnet 1a**
```
aws ec2 associate-route-table --route-table-id rtb-1b2c3d4e5f6g7h8i --subnet-id subnet-2c3d4e5f6g7h8i9j --region us-east-1
```
**Description:** Associates the private route table with the first private subnet.

- **Associate private route table with private subnet 1b**
```
aws ec2 associate-route-table --route-table-id rtb-1b2c3d4e5f6g7h8i --subnet-id subnet-3d4e5f6g7h8i9j0k --region us-east-1
```
**Description:** Associates the private route table with the second private subnet.

#### Cluster creation and IAM roles (hardcoded ARNs and names)

- **Write cluster trust policy file**
```
cat > /tmp/cluster-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"eks.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
```
**Description:** Creates a local JSON file with the trust policy for the EKS cluster role.

- **Create EKS cluster IAM role**
```
aws iam create-role --role-name eliezer-eks-cluster-role --assume-role-policy-document file:///tmp/cluster-trust.json --tags Key=env,Value=training Key=team,Value=academy-sre Key=student,Value=eliezer
```
**Description:** Creates the IAM role that EKS will assume for cluster control plane operations.

- **Attach AmazonEKSClusterPolicy to cluster role**
```
aws iam attach-role-policy --role-name eliezer-eks-cluster-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```
**Description:** Grants the role the managed policy required for EKS cluster operations.

- **Create EKS cluster**
```
aws eks create-cluster --name academy-eliezer-cluster --region us-east-1 --kubernetes-version 1.35 --role-arn arn:aws:iam::123456789012:role/eliezer-eks-cluster-role --resources-vpc-config subnetIds=subnet-0a1b2c3d4e5f6g7h,subnet-1b2c3d4e5f6g7h8i,subnet-2c3d4e5f6g7h8i9j,subnet-3d4e5f6g7h8i9j0k --access-config authenticationMode=API_AND_CONFIG_MAP --tags env=training,team=academy-sre,student=eliezer
```
**Description:** Creates the EKS control plane in the specified VPC subnets with the given role and Kubernetes version.

- **Wait for cluster to become ACTIVE**
```
aws eks wait cluster-active --name "academy-eliezer-cluster" --region "us-east-1"
```
**Description:** Blocks until the EKS cluster reaches the ACTIVE state.

- **Describe cluster (summary)**
```
aws eks describe-cluster --name academy-eliezer-cluster --region us-east-1 --query "cluster.{Name:name,Status:status,Version:version,Endpoint:endpoint}" --output table
```
**Description:** Shows a concise table with cluster name, status, version, and endpoint.

#### Addons and EBS CSI IRSA role

- **Associate OIDC provider with cluster (eksctl)**
```
eksctl utils associate-iam-oidc-provider --cluster academy-eliezer-cluster --region us-east-1 --approve
```
**Description:** Registers the cluster OIDC provider in IAM to enable IRSA (IAM Roles for Service Accounts).

- **Create EBS CSI IRSA trust policy file (example uses account 123456789012 and OIDC id placeholder)**
```
cat > /tmp/ebs-irsa-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLEOIDCID"},
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLEOIDCID:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLEOIDCID:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF
```
**Description:** Writes the trust policy JSON for the EBS CSI driver IRSA role (replace `EXAMPLEOIDCID` with the real OIDC ID).

- **Create EBS CSI IRSA IAM role**
```
aws iam create-role --role-name eliezer-ebs-csi-irsa-role --assume-role-policy-document file:///tmp/ebs-irsa-trust.json --tags Key=env,Value=training Key=team,Value=academy-sre Key=student,Value=eliezer
```
**Description:** Creates the IAM role that the EBS CSI driver service account will assume.

- **Attach AmazonEBSCSIDriverPolicy to IRSA role**
```
aws iam attach-role-policy --role-name eliezer-ebs-csi-irsa-role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```
**Description:** Grants the IRSA role the permissions required by the EBS CSI driver.

- **Create or enable VPC CNI addon**
```
aws eks create-addon --cluster-name academy-eliezer-cluster --addon-name vpc-cni --region us-east-1
```
**Description:** Installs or updates the AWS VPC CNI addon in the cluster.

- **Create or enable CoreDNS addon**
```
aws eks create-addon --cluster-name academy-eliezer-cluster --addon-name coredns --region us-east-1
```

**Description:** Installs or updates CoreDNS in the cluster.

- **Create or enable kube-proxy addon**
```
aws eks create-addon --cluster-name academy-eliezer-cluster --addon-name kube-proxy --region us-east-1
```
**Description:** Installs or updates kube-proxy in the cluster.

- **Create or enable AWS EBS CSI driver addon with IRSA role**
```
aws eks create-addon --cluster-name academy-eliezer-cluster --addon-name aws-ebs-csi-driver --region us-east-1 --service-account-role-arn arn:aws:iam::123456789012:role/eliezer-ebs-csi-irsa-role
```
**Description:** Installs the EBS CSI driver and associates it with the IRSA role.

- **List installed addons**
```
aws eks list-addons --cluster-name academy-eliezer-cluster --region us-east-1 --output table
```
**Description:** Lists the addons currently installed on the cluster.

#### Node group (worker nodes)

- **Write node trust policy file**
```
cat > /tmp/node-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
```
**Description:** Creates the trust policy JSON for the EC2 node IAM role.

- **Create node IAM role**
```
aws iam create-role --role-name eliezer-eks-node-role --assume-role-policy-document file:///tmp/node-trust.json --tags Key=env,Value=training Key=team,Value=academy-sre Key=student,Value=eliezer
```
**Description:** Creates the IAM role that EC2 worker nodes will assume.

- **Attach AmazonEKSWorkerNodePolicy to node role**
```
aws iam attach-role-policy --role-name eliezer-eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
```
**Description:** Grants node role permissions required by EKS worker nodes.

- **Attach AmazonEKS_CNI_Policy to node role**
```
aws iam attach-role-policy --role-name eliezer-eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```
**Description:** Grants CNI-related permissions to the node role.

- **Attach AmazonEC2ContainerRegistryReadOnly to node role**
```
aws iam attach-role-policy --role-name eliezer-eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```
**Description:** Allows nodes to pull images from ECR.

- **Create EKS managed node group**
```
aws eks create-nodegroup --cluster-name academy-eliezer-cluster --nodegroup-name academy-eliezer-ng-01 --region us-east-1 --node-role arn:aws:iam::123456789012:role/eliezer-eks-node-role --subnets subnet-2c3d4e5f6g7h8i9j subnet-3d4e5f6g7h8i9j0k --instance-types c7i-flex.large --ami-type AL2_x86_64 --capacity-type ON_DEMAND --scaling-config minSize=0,maxSize=3,desiredSize=2 --tags env=training,team=academy-sre,student=eliezer
```
**Description:** Creates a managed node group with the specified instance type, subnets, scaling config, and IAM role.

- **Wait for node group to become ACTIVE**
```
aws eks wait nodegroup-active --cluster-name "academy-eliezer-cluster" --nodegroup-name "academy-eliezer-ng-01" --region "us-east-1"
```
**Description:** Blocks until the node group is active and ready.

- **Describe node group (summary)**
```
aws eks describe-nodegroup --cluster-name academy-eliezer-cluster --nodegroup-name academy-eliezer-ng-01 --region us-east-1 --query "nodegroup.{Name:nodegroupName,Status:status,Version:version,DesiredSize:scalingConfig.desiredSize}" --output table
```
**Description:** Shows node group name, status, Kubernetes version, and desired size.

#### Access and verification (kubectl / kubeconfig)

- **Update kubeconfig for the cluster**
```
aws eks update-kubeconfig --region us-east-1 --name academy-eliezer-cluster
```
**Description:** Writes cluster credentials and endpoint into `~/.kube/config` for `kubectl` access.

- **Show current kubectl context**
```
kubectl config current-context
```
**Description:** Prints the current kubectl context name.

- **List cluster nodes**
```
kubectl get nodes -o wide
```
**Description:** Lists worker nodes with additional details (IP, instance type, etc.).

- **List all pods in all namespaces**
```
kubectl get pods -A
```
**Description:** Shows all pods across all namespaces for cluster verification.

- **Describe cluster summary via AWS CLI**
```
aws eks describe-cluster --name academy-eliezer-cluster --region us-east-1 --query "cluster.{Name:name,Status:status,Version:version}" --output table
```
**Description:** Quick AWS-side summary of cluster name, status, and version.

- **Describe node group summary via AWS CLI*
```
aws eks describe-nodegroup --cluster-name academy-eliezer-cluster --nodegroup-name academy-eliezer-ng-01 --region us-east-1 --query "nodegroup.{Name:nodegroupName,Status:status,Version:version,DesiredSize:scalingConfig.desiredSize}" --output table
```

**Description:** Quick AWS-side summary of the managed node group.

- **List addons via AWS CLI**
```
aws eks list-addons --cluster-name academy-eliezer-cluster --region us-east-1 --output table
```

**Description:** Lists installed EKS addons for the cluster.

If you want, I can now:

- Remove the placeholder IDs and replace them with the actual IDs from your account (I can show the exact commands to capture those IDs), or
    
- Produce a compact, ordered list of only the `aws`/`eksctl`/`kubectl` commands (no `cat` or `describe` helper commands) ready to run in sequence. Which would you prefer?