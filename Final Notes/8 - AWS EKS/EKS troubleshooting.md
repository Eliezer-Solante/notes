
filename - | tee -a eliezer-eks-troubleshooting.txt

aws iam detach-role-policy \
  --role-name eliezer-eks-node-role\
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy


aws ec2 describe-instances --filters "Name=tag:eks:nodegroup-name,Values=academy-eliezer-ng-01" "Name=tag:kubernetes.io/cluster/academy-eliezer-cluster,Values=owned" --query "Reservations[*].Instances[*].InstanceId" --output text

kubectl get nodes
aws iam list-attached-role-policies --role-name eliezer-eks-node-role
aws eks describe-nodegroup --cluster-name academy-eliezer-cluster --nodegroup-name academy-eliezer-ng-01 --query 'nodegroup.health'

Detatching the AmazonEKSworkerNodePolicy did not prevent nodes from joining or reaching `Ready` state