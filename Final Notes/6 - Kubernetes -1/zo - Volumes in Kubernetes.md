![[Pasted image 20260820154954.png]]

![[Pasted image 20260820155127.png]]

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - image: kodekloud/event-simulator
    name: event-simulator
    resources: {}
    env:
    - name: LOG_HANDLERS
      value: file
    volumeMounts:
    - mountPath: /log
      name: log-volume
  
  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/webapp
      type: Directory
status: {}
```