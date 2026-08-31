```
aws eks describe-cluster \
  --name academy-eliezer-cluster \
  --region us-east-1 \
  --query "cluster.{Name:name,Version:version,Status:status}" \
  --output table | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

```
kubectl get nodes -o wide | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

```
aws eks list-addons \
  --cluster-name academy-eliezer-cluster \
  --region us-east-1 \
  --output table | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

```
kubectl get pods -l app=nginx-upgrade-test -o wide | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```