
The practical assessment will take place on KodeKloud's AWS EKS Playground. You will bootstrap the EKS lab using the playground's AWS CloudShell.

1. Using Chrome browser, log in to your KodeKloud account.
    
2. Go to [https://kodekloud.com/cloud-playgrounds/aws](https://kodekloud.com/cloud-playgrounds/aws)
    
3. Log in to the AWS console using the given IAM username and password.
    
4. Open CloudShell and download `eks-bootstrap`. Run the `eks-bootstrap` script to provision the broken EKS cluster. Cluster creation takes about 15 minutes.
    
    ```shell
    wget https://aws-assessment-resources.ap-south-1.linodeobjects.com/aws-eks-v2/eks-bootstrap
    chmod +x eks-bootstrap
    ./eks-bootstrap
    ```
    
5. Do not run `instance-reboot` and `deploy-backend-db` unless explicitly instructed on specific questions.

# AWS EKS Assessment — Full Summary

Score on last submission: **78/100** (passing = 85/100) Root cause of failure: Task 4 approach (see "Fixes Applied" section at the end).

---

## Setup

```bash
wget https://aws-assessment-resources.ap-south-1.linodeobjects.com/aws-eks-v2/eks-bootstrap
chmod +x eks-bootstrap
./eks-bootstrap
```

Note: `instance-reboot` and `deploy-backend-db` are only run when a specific task calls for them (Tasks 9 and 11).

---

## Task 1 — Inspect node status

**Objective:** Retrieve node status (internal/external IP, OS image, kernel version, container runtime).

```bash
kubectl get nodes -o wide > Q1.txt
cat Q1.txt
```

**Result:** PASSED (5/5)

---

## Task 2 — Inspect deployments/pods

**Objective:** Identify root cause of pod failures.

```bash
# 1. All pod status
kubectl get pods -A -o wide > Q2-1.txt

# 2. Describe one sre-app pod
kubectl get pods -A | grep sre-app
kubectl describe pod <sre-app-pod-name> -n <namespace> > Q2-2.txt

# 3. Describe one devops-app pod
kubectl get pods -A | grep devops-app
kubectl describe pod <devops-app-pod-name> -n <namespace> > Q2-3.txt
```

**Result:** PASSED (5/5 each — Q2-1, Q2-2, Q2-3)

---

## Task 3 — Diagnose and fix `NotReady` nodes

**Objective:** Root-cause and fix `NotReady` node status prior to remediation.

```bash
script Q3.txt

# Pre-fix
kubectl get nodes -o wide
kubectl describe node ip-172-31-41-219.ec2.internal

# Check inside node
aws ssm start-session --target i-010d0a34ae3f0cbf0
sudo systemctl status kubelet
sudo systemctl status containerd
sudo journalctl -u kubelet -n 100 --no-pager

# Fix
sudo systemctl restart containerd
sudo systemctl restart kubelet
sudo systemctl status kubelet
sudo systemctl status containerd
exit

# Post-fix
kubectl get nodes -o wide
kubectl describe node ip-172-31-41-219.ec2.internal

exit
```

**Result:** FAILED (6/8) — pre-fix (3/3) and fix command (3/3) scored fully, but **post-fix confirmation (0/2)** was marked incomplete.

### Fix applied

The post-fix section of `Q3.txt` needs explicit, unambiguous evidence the node returned to `Ready`. Re-run and re-capture cleanly:

```bash
{
  echo "=== POST-FIX: kubectl get nodes -o wide ==="
  kubectl get nodes -o wide
  echo ""
  echo "=== POST-FIX: kubectl describe node ip-172-31-41-219.ec2.internal ==="
  kubectl describe node ip-172-31-41-219.ec2.internal
} >> Q3.txt
```

Confirm the `STATUS` column shows `Ready` and the `describe node` **Conditions** section shows `Ready: True` with a recent `LastHeartbeatTime` — both need to be visible in the saved file.

---

## Task 4 — Schedule `sre-app` and `devops-app` to correct nodes

**Objective:** Resolve `Pending` status by assigning deployments to `alias=Node-1` / `alias=Node-2`.

### What went wrong originally

`sre-app` had a pre-existing **required nodeAffinity** (`nodetype In (sre, production-failover)`), and `devops-app` had a pre-existing **nodeSelector** (`nodetype: devops`). The original (incorrect) fix **deleted** these requirements instead of **satisfying** them by labeling the nodes. This caused Q4-1, Q4-2, Q5-1, Q5-2 to fail, since the grader expected the `nodetype` labels to appear on the nodes.

### Corrected commands

```bash
# Find node names
kubectl get nodes -l alias=Node-1
kubectl get nodes -l alias=Node-2

# Label nodes to satisfy each deployment's original requirement
kubectl label node <node-1-actual-name> nodetype=sre
kubectl label node <node-2-actual-name> nodetype=devops

# Restore/set correct nodeSelectors on each deployment
kubectl patch deployment sre-app -n sre-app -p '{"spec":{"template":{"spec":{"nodeSelector":{"alias":"Node-1","nodetype":"sre"}}}}}'
kubectl patch deployment devops-app -n devops-app -p '{"spec":{"template":{"spec":{"nodeSelector":{"alias":"Node-2","nodetype":"devops"}}}}}'

# Verify
kubectl get pods -n sre-app -o wide
kubectl get pods -n devops-app -o wide

# Save deliverables
kubectl describe node <node-1-actual-name> > Q4-1.txt
kubectl describe node <node-2-actual-name> > Q4-2.txt
```

**Original result:** FAILED (0/5 each — Q4-1, Q4-2) **Fix:** Labels `nodetype=sre` / `nodetype=devops` now appear on the respective nodes, matching grader expectations.

---

## Task 5 — Verify `sre-app` / `devops-app` pods running

**Objective:** Confirm pods running and correctly assigned.

```bash
kubectl get pods -n sre-app -o wide
kubectl get pods -n devops-app -o wide

kubectl describe pod <sre-app-pod-name> -n sre-app > Q5-1.txt
kubectl describe pod <devops-app-pod-name> -n devops-app > Q5-2.txt
```

**Original result:** FAILED (0/5 each — Q5-1, Q5-2) — direct consequence of the Task 4 issue above. **Fix:** Once Task 4's nodeSelectors/labels are corrected and pods reschedule onto Node-1/Node-2 as `Running`, re-run `describe pod` and re-save. The **Status: Running** and **Node:** fields in these files will now match what the grader expects.

---

## Task 6 — Label node with custom label

**Objective:** Add `nodetype=production-failover` to node `alias=Node-3`.

```bash
kubectl get nodes -l alias=Node-3
kubectl label node <node-3-actual-name> nodetype=production-failover
kubectl describe node <node-3-actual-name> > Q6.txt
```

**Result:** PASSED (5/5)

---

## Task 7 — Cordon Node-1 and evict `sre-app` pods

**Objective:** Make `alias=Node-1` unschedulable and evict its `sre-app` pods.

```bash
script Q7-1.txt

kubectl cordon <node-1-actual-name>

# PDB blocked eviction (Allowed disruptions: 0) — removed to permit eviction
kubectl get pdb -n sre-app
kubectl delete pdb sre-app -n sre-app

kubectl drain <node-1-actual-name> --pod-selector=app=sre-app --ignore-daemonsets --delete-emptydir-data

exit
```

```bash
kubectl describe node <node-1-actual-name> > Q7-2.txt
```

**Result:** PASSED (5/5 each — Q7-1, Q7-2)

---

## Task 8 — Verify pod migration to Node-3

**Objective:** Confirm `sre-app` pods rescheduled onto `alias=Node-3`.

```bash
kubectl get pods -n sre-app -o wide
kubectl get nodes -l alias=Node-3 --show-labels

# sre-app selector updated to match Node-3's nodetype label
kubectl patch deployment sre-app -n sre-app -p '{"spec":{"template":{"spec":{"nodeSelector":{"alias":null,"nodetype":"production-failover"}}}}}'

kubectl get pods -n sre-app -o wide

kubectl get nodes -l alias=Node-3 -o name
kubectl describe node <node-3-actual-name> > Q8.txt
```

**Result:** PASSED (5/5)

---

## Task 9 — Reboot Node-1, delete Node-3, reschedule back

**Objective:** Simulate reboot, delete Node-3, move `sre-app` back to `alias=Node-1`.

```bash
wget https://aws-assessment-resources.ap-south-1.linodeobjects.com/aws-eks-v2/instance-reboot
chmod +x instance-reboot
./instance-reboot
# → Rebooting i-03dd4ca4a3d8a152c (ip-172-31-26-113.ec2.internal)

kubectl get nodes -o wide
kubectl get nodes -l alias=Node-1
kubectl uncordon ip-172-31-26-113.ec2.internal

kubectl get nodes -l alias=Node-3
kubectl delete node <node-3-actual-name>

kubectl patch deployment sre-app -n sre-app -p '{"spec":{"template":{"spec":{"nodeSelector":{"nodetype":null,"alias":"Node-1"}}}}}'

kubectl get pods -n sre-app -o wide
```

```bash
kubectl describe node ip-172-31-26-113.ec2.internal > Q9-1.txt
kubectl get pods -n sre-app -o wide > Q9-2.txt
```

**Result:** PASSED (3/3 each — Q9-1, Q9-2)

---

## Task 10 — Resource utilization (metrics)

**Objective:** Retrieve CPU/Memory metrics for nodes and both deployments.

```bash
# metrics-server was already installed/running
kubectl top nodes > Q10-1.txt
kubectl top pods -n sre-app > Q10-2.txt
kubectl top pods -n devops-app > Q10-3.txt
```

**Result:** PASSED (5/5, 3/3, 3/3)

---

## Task 11 — Fix broken `backend-db` deployment + storage

**Objective:** Fix PV/PVC binding and get `backend-db` pods running.

```bash
wget https://aws-assessment-resources.ap-south-1.linodeobjects.com/aws-eks-v2/deploy-backend-db
chmod +x deploy-backend-db
./deploy-backend-db

kubectl get pv
kubectl get pvc -A
kubectl get deploy -A | grep backend-db

kubectl get pv backend-db-pv -o yaml
kubectl get pvc backend-db-pvc -n backend-db -o yaml
```

**Issue 1 — accessMode mismatch:** PV offered `ReadWriteMany`, PVC required `ReadWriteOnce`.

```bash
kubectl get pv backend-db-pv -o yaml > pv-backup.yaml
sed -i 's/ReadWriteMany/ReadWriteOnce/' pv-backup.yaml
sed -i '/resourceVersion:/d; /uid:/d; /creationTimestamp:/d' pv-backup.yaml
kubectl delete pv backend-db-pv
kubectl apply -f pv-backup.yaml

kubectl get pv
kubectl get pvc -n backend-db
```

**Issue 2 — wrong claimName in deployment volume spec:** deployment referenced `backend-pvc-db` (typo) instead of `backend-db-pvc`.

```bash
kubectl patch deployment backend-db -n backend-db --type=json -p '[{"op":"replace","path":"/spec/template/spec/volumes/0/persistentVolumeClaim/claimName","value":"backend-db-pvc"}]'

kubectl get pods -n backend-db -o wide
```

```bash
kubectl describe pv backend-db-pv > Q11-1.txt
kubectl describe pvc backend-db-pvc -n backend-db > Q11-2.txt
kubectl describe deployment backend-db -n backend-db > Q11-3.txt
```

**Result:** PASSED (5/5, 3/3, 3/3)

---

## Fixes Applied (Q3–Q5)

|File|Original|Root cause|Corrective action|
|---|---|---|---|
|Q3.txt|6/8|Post-fix confirmation not clearly captured|Re-append explicit post-fix `get nodes -o wide` + `describe node` output showing `Ready: True`|
|Q4-1.txt|0/5|Deleted `nodetype` requirement instead of labeling node|Labeled Node-1 with `nodetype=sre`, restored nodeSelector on `sre-app`|
|Q4-2.txt|0/5|Deleted `nodetype` requirement instead of labeling node|Labeled Node-2 with `nodetype=devops`, restored nodeSelector on `devops-app`|
|Q5-1.txt|0/5|Consequence of Q4-1 fix approach|Re-described `sre-app` pod once correctly scheduled on Node-1|
|Q5-2.txt|0/5|Consequence of Q4-2 fix approach|Re-described `devops-app` pod once correctly scheduled on Node-2|

**Note:** This submission was scored as a one-time attempt, so these corrections could not be resubmitted for the graded run. This summary documents the correct approach for a retake or future reference.



![[Pasted image 20260901165518.png]]