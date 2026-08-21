![[Pasted image 20260820155446.png]]

![[Pasted image 20260820155610.png]]

![[Pasted image 20260820155626.png]]

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 100Mi
  hostPath:
    path: /pv/log
  persistentVolumeReclaimPolicy: Retain                                       
```
