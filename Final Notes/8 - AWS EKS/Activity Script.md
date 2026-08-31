
```bash
./eliezer-eks-cluster-upgrade-part1.sh identity   2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh vpc        2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh cluster    2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh addons     2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh nodegroup  2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh access     2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
./eliezer-eks-cluster-upgrade-part1.sh verify     2>&1 | tee -a aws-eks-cluster-upgrade-part1-eliezer.txt
```

