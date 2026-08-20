![[Pasted image 20260820155755.png]]
![[Pasted image 20260820155812.png]]

![[Pasted image 20260820155821.png]]
![[Pasted image 20260820160004.png]]


![[Pasted image 20260820160057.png]]

![[Pasted image 20260820160159.png]]

![[Pasted image 20260820160217.png]]

![[Pasted image 20260820160339.png]]

Once you create a PVC use it in a POD definition file by specifying the PVC Claim name under persistentVolumeClaim section in the volumes section like this:

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

The same is true for ReplicaSets or Deployments. Add this to the pod template section of a Deployment on ReplicaSet.

Reference URL: [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes)