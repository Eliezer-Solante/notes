
filename - | tee -a eliezer-eks-troubleshooting.txt

aws iam detach-role-policy \
  --role-name eliezer-eks-node-role\
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy


aws ec2 describe-instances --filters "Name=tag:eks:nodegroup-name,Values=academy-eliezer-ng-01" "Name=tag:kubernetes.io/cluster/academy-eliezer-cluster,Values=owned" --query "Reservations[*].Instances[*].InstanceId" --output text