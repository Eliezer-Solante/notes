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


The AWS CLI supports `--query` and `--output`, so you can filter results cleanly.

- **Amazon VPC CNI (latest only)**
    

bash

```
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.36 \
  --query "addons[0].addonVersions[-1].addonVersion" \
  --output text \
  --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

- **CoreDNS (latest only)**
    

bash

```
aws eks describe-addon-versions \
  --addon-name coredns \
  --kubernetes-version 1.36 \
  --query "addons[0].addonVersions[-1].addonVersion" \
  --output text \
  --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

- **Amazon EBS CSI Driver (latest only)**
    

bash

```
aws eks describe-addon-versions \
  --addon-name aws-ebs-csi-driver \
  --kubernetes-version 1.36 \
  --query "addons[0].addonVersions[-1].addonVersion" \
  --output text \
  --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

- **kube-proxy (latest only)**
    

bash

```
aws eks describe-addon-versions \
  --addon-name kube-proxy \
  --kubernetes-version 1.36 \
  --query "addons[0].addonVersions[-1].addonVersion" \
  --output text \
  --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```




### 🔧 Update commands for each add-on

- **Amazon VPC CNI**
    

bash

```
aws eks update-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name vpc-cni \
  --addon-version v1.19.0-eksbuild.1 \
  --region us-east-1
```

- **CoreDNS**
    

bash

```
aws eks update-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name coredns \
  --addon-version v1.13.1-eksbuild.1 \
  --region us-east-1
```

- **Amazon EBS CSI Driver**
    

bash

```
aws eks update-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name aws-ebs-csi-driver \
  --addon-version v1.42.0-eksbuild.1 \
  --region us-east-1
```

- **kube-proxy**
    

bash

```
aws eks update-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name kube-proxy \
  --addon-version v1.33.0-eksbuild.2 \
  --region us-east-1
```



### 🔧 Step 2 — Check status of each add‑on

Replace `<addon-name>` with each add‑on from the list:

- **Amazon VPC CNI**
    

bash

```
aws eks describe-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name vpc-cni \
  --region us-east-1 \
  --query "addon.{Name:addonName,Version:addonVersion,Status:status}" \
  --output table | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

- **CoreDNS**
    

bash

```
aws eks describe-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name coredns \
  --region us-east-1 \
  --query "addon.{Name:addonName,Version:addonVersion,Status:status}" \
  --output table | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

- **Amazon EBS CSI Driver**
    

bash

```
aws eks describe-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1 \
  --query "addon.{Name:addonName,Version:addonVersion,Status:status}" \
  --output table | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```

- **kube-proxy**
    

bash

```
aws eks describe-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name kube-proxy \
  --region us-east-1 \
  --query "addon.{Name:addonName,Version:addonVersion,Status:status}" \
  --output table | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt
```






aws eks update-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name vpc-cni \
  --addon-version v1.23.0-eksbuild.1 \
  --resolve-conflicts PRESERVE \
  --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt


aws eks update-addon \
  --cluster-name academy-eliezer-cluster \
  --addon-name coredns \
  --addon-version v1.14.3-eksbuild.14 \
  --resolve-conflicts PRESERVE \
  --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt


  aws eks update-addon --cluster-name academy-eliezer-cluster --addon-name kube-proxy --addon-version v1.36.0-eksbuild.17 --resolve-conflicts PRESERVE --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt

  aws eks update-addon --cluster-name academy-eliezer-cluster --addon-name aws-ebs-csi-driver --addon-version v1.65.0-eksbuild.1 --service-account-role-arn arn:aws:iam::687259231807:role/eliezer-ebs-csi-irsa-role --resolve-conflicts PRESERVE --region us-east-1 | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt




  aws eks list-addons --cluster-name academy-eliezer-cluster --region us-east-1 --output table



  aws eks describe-cluster \
  --name academy-eliezer-cluster \
  --region us-east-1 \
  --query "cluster.{Name:name,Version:version,Status:status}" \
  --output table | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt


  kubectl get nodes -o wide | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt


  aws eks list-addons \
  --cluster-name academy-eliezer-cluster \
  --region us-east-1 \
  --output table  | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt


  kubectl get pods -l app=nginx-upgrade-test -o wide | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt

  kubectl get pods -l app=nginx-upgrade-test | tee -a aws-eks-cluster-upgrade-part2-eliezer.txt