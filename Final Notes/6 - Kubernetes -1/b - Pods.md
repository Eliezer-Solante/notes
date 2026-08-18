![[Pasted image 20260813105351.png]]

![[Pasted image 20260813105048.png]]
![[Pasted image 20260813105556.png]]

![[Pasted image 20260813110127.png]]
![[Pasted image 20260813110113.png]]

user `--dry-run -o yaml` to see how you imperative command will look like in a yaml file
![[Pasted image 20260817133428.png]]
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

to delete pod immediately
```bash
--force --grace-period=0
```