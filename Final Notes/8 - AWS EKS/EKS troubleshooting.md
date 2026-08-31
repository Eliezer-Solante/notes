
filename - | tee -a eliezer-eks-troubleshooting.txt

aws iam detach-role-policy \
  --role-name eliezer-eks-node-role\
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy


