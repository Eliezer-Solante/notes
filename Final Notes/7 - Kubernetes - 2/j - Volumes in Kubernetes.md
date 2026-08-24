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


```yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
  name: webserver
spec:
  initContainers:
  - image: ubuntu:latest
    name: sidecar-container
    command: ["sh", "-c", "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"]
    volumeMounts:
    - mountPath: /var/log/nginx
      name: shared-logs
    restartPolicy: Always
  containers:
  - image: nginx:latest
    name: nginx-container
    resources: {}
    volumeMounts:
    - mountPath: /var/log/nginx
      name: shared-logs

  volumes:
  - name: shared-logs
    emptyDir: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```