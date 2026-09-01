
filename - | tee -a eliezer-eks-troubleshooting.txt

aws iam detach-role-policy \
  --role-name eliezer-eks-node-role\
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy


aws ec2 describe-instances --filters "Name=tag:eks:nodegroup-name,Values=academy-eliezer-ng-01" "Name=tag:kubernetes.io/cluster/academy-eliezer-cluster,Values=owned" --query "Reservations[*].Instances[*].InstanceId" --output text

kubectl get nodes

aws iam list-attached-role-policies --role-name eliezer-eks-node-role

aws eks describe-nodegroup --cluster-name academy-eliezer-cluster --nodegroup-name academy-eliezer-ng-01 --query 'nodegroup.health'

Detatching the AmazonEKSworkerNodePolicy did not prevent nodes from joining or reaching `Ready` state

```
=== SCENARIO 1: NODE IAM / CLUSTER AUTH ===
```
```
=== SCENARIO 2: CONTROL PLANE-TO-NODE NETWORKING ===
```
```
=== SCENARIO 3: DNS (COREDNS) ===
```
```
=== SCENARIO 4: STORAGE (EBS CSI DRIVER) ===
```
```
=== SCENARIO 5: OUTBOUND NETWORKING (NAT GATEWAY ROUTE) ===
```
```
=== SCENARIO 6: SCHEDULING (TAINTS AND TOLERATIONS) ===
```
```
=== CLEAN UP ===
```





Prerequisites
You need a running EKS cluster with a node group and the standard add-ons (VPC CNI, CoreDNS, kube-proxy, Amazon EBS CSI Driver) before starting.

If you came straight from EKS Cluster Upgrade and chose to leave your cluster running: use it as-is, no redeploy needed.
If you already cleaned it up: redeploy it via CLI following the same steps from EKS Deployment Basics / EKS Cluster Upgrade Part 1.
You'll also need a persistent workload and a PVC-backed pod running before Scenario 4, so deploy the following now if it's not already there:

aws ec2 describe-security-groups --group-ids sg-012b6ae1ce69accde --region us-east-1 --query 'SecurityGroups[0].IpPermissions'

aws ec2 authorize-security-group-ingress --group-id sg-012b6ae1ce69accde --region us-east-1 --protocol -1 --source-group sg-012b6ae1ce69accde