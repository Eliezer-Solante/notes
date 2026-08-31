
Perfect — let’s go back to the **command list** and expand it with the **available options** for each AWS CLI command. This way you’ll know not only the exact command but also what flags/parameters you can use when customizing.

### Identity commands

- **aws sts get-caller-identity** Options: none (just returns `UserId`, `Account`, `Arn`).
    
- **aws --version** Options: none (prints CLI version).
    
- **eksctl version** Options: none (prints eksctl version).
    
- **aws eks describe-cluster-versions** Options:
    
    - `--region <region>`
        
    - `--query <JMESPath>`
        
    - `--output <json|table|text>`
        

### VPC and networking

- **aws ec2 create-vpc** Options:
    
    - `--cidr-block <CIDR>`
        
    - `--region <region>`
        
    - `--tag-specifications <tags>`
        
- **aws ec2 describe-vpcs** Options:
    
    - `--filters <filters>`
        
    - `--query <JMESPath>`
        
    - `--output <format>`
        
- **aws ec2 modify-vpc-attribute** Options:
    
    - `--vpc-id <id>`
        
    - `--enable-dns-hostnames '{"Value":true}'`
        
    - `--enable-dns-support '{"Value":true}'`
        
- **aws ec2 create-subnet** Options:
    
    - `--vpc-id <id>`
        
    - `--cidr-block <CIDR>`
        
    - `--availability-zone <az>`
        
    - `--tag-specifications <tags>`
        
- **aws ec2 describe-subnets** Options:
    
    - `--filters <filters>`
        
    - `--query <JMESPath>`
        
    - `--output <format>`
        
- **aws ec2 modify-subnet-attribute** Options:
    
    - `--subnet-id <id>`
        
    - `--map-public-ip-on-launch`
        
- **aws ec2 create-internet-gateway** Options:
    
    - `--tag-specifications <tags>`
        
- **aws ec2 attach-internet-gateway** Options:
    
    - `--internet-gateway-id <id>`
        
    - `--vpc-id <id>`
        
- **aws ec2 allocate-address** Options:
    
    - `--domain vpc`
        
    - `--tag-specifications <tags>`
        
- **aws ec2 create-nat-gateway** Options:
    
    - `--subnet-id <id>`
        
    - `--allocation-id <eipalloc>`
        
    - `--tag-specifications <tags>`
        
- **aws ec2 describe-nat-gateways** Options:
    
    - `--filter <filters>`
        
    - `--query <JMESPath>`
        
    - `--output <format>`
        
- **aws ec2 wait nat-gateway-available** Options:
    
    - `--nat-gateway-ids <ids>`
        
- **aws ec2 create-route-table** Options:
    
    - `--vpc-id <id>`
        
    - `--tag-specifications <tags>`
        
- **aws ec2 create-route** Options:
    
    - `--route-table-id <id>`
        
    - `--destination-cidr-block <CIDR>`
        
    - `--gateway-id <id>` OR `--nat-gateway-id <id>`
        
- **aws ec2 associate-route-table** Options:
    
    - `--route-table-id <id>`
        
    - `--subnet-id <id>`
        

### Cluster creation

- **aws iam create-role** Options:
    
    - `--role-name <name>`
        
    - `--assume-role-policy-document file://<path>`
        
    - `--tags <tags>`
        
- **aws iam attach-role-policy** Options:
    
    - `--role-name <name>`
        
    - `--policy-arn <arn>`
        
- **aws eks create-cluster** Options:
    
    - `--name <cluster>`
        
    - `--region <region>`
        
    - `--kubernetes-version <version>`
        
    - `--role-arn <arn>`
        
    - `--resources-vpc-config subnetIds=<ids>`
        
    - `--access-config authenticationMode=<mode>`
        
    - `--tags <tags>`
        
- **aws eks wait cluster-active** Options:
    
    - `--name <cluster>`
        
    - `--region <region>`
        
- **aws eks describe-cluster** Options:
    
    - `--name <cluster>`
        
    - `--region <region>`
        
    - `--query <JMESPath>`
        
    - `--output <format>`
        

### Addons

- **eksctl utils associate-iam-oidc-provider** Options:
    
    - `--cluster <name>`
        
    - `--region <region>`
        
    - `--approve`
        
- **aws eks create-addon** Options:
    
    - `--cluster-name <name>`
        
    - `--addon-name <addon>`
        
    - `--region <region>`
        
    - `--service-account-role-arn <arn>` (only for EBS CSI driver)
        
- **aws eks list-addons** Options:
    
    - `--cluster-name <name>`
        
    - `--region <region>`
        
    - `--output <format>`
        

### Nodegroup

- **aws eks create-nodegroup** Options:
    
    - `--cluster-name <name>`
        
    - `--nodegroup-name <name>`
        
    - `--region <region>`
        
    - `--node-role <arn>`
        
    - `--subnets <ids>`
        
    - `--instance-types <types>`
        
    - `--ami-type <ami>`
        
    - `--capacity-type <ON_DEMAND|SPOT>`
        
    - `--scaling-config minSize=,maxSize=,desiredSize=`
        
    - `--tags <tags>`
        
- **aws eks wait nodegroup-active** Options:
    
    - `--cluster-name <name>`
        
    - `--nodegroup-name <name>`
        
    - `--region <region>`
        
- **aws eks describe-nodegroup** Options:
    
    - `--cluster-name <name>`
        
    - `--nodegroup-name <name>`
        
    - `--region <region>`
        
    - `--query <JMESPath>`
        
    - `--output <format>`


### Access & verification (kubectl / kubeconfig)

- **aws eks update-kubeconfig** Options:
    
    - `--region <region>`
        
    - `--name <cluster>`
        
    - `--role-arn <arn>` (optional, if assuming a role)
        
    - `--alias <context-name>` (optional, sets custom context name)
        
    - `--profile <profile>` (optional, use a specific AWS CLI profile)
        
- **kubectl config current-context** Options: none (just prints the current context).
    
- **kubectl get nodes** Options:
    
    - `-o wide` → show extra details (IP, instance type, etc.)
        
    - `-o json` / `-o yaml` → structured output
        
    - `--selector <label>` → filter nodes by label
        
    - `--all-namespaces` → not applicable here (nodes are cluster-wide)
        
- **kubectl get pods** Options:
    
    - `-A` or `--all-namespaces` → list pods across all namespaces
        
    - `-n <namespace>` → restrict to a specific namespace
        
    - `-o wide` → show extra details (node, IP, etc.)
        
    - `-o json` / `-o yaml` → structured output
        
    - `--selector <label>` → filter pods by label
        

### Verification commands

- **aws eks describe-cluster** Options:
    
    - `--name <cluster>`
        
    - `--region <region>`
        
    - `--query <JMESPath>` (filter output fields)
        
    - `--output <json|table|text|yaml>`
        
- **aws eks describe-nodegroup** Options:
    
    - `--cluster-name <name>`
        
    - `--nodegroup-name <name>`
        
    - `--region <region>`
        
    - `--query <JMESPath>`
        
    - `--output <format>`
        
- **aws eks list-addons** Options:
    
    - `--cluster-name <name>`
        
    - `--region <region>`
        
    - `--output <format>`