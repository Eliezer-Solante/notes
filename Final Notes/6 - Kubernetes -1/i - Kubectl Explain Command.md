![[Pasted image 20260813183047.png]]

![[Pasted image 20260813183148.png]]
![[Pasted image 20260813183219.png]]
![[Pasted image 20260813183238.png]]


<mark style="background: #FFF3A3A6;">to search for anything on resources</mark>( in here only info about these  nodes|pods|services|deployments )
```bash
kubectl api-resources | grep -E 'nodes|pods|services|deployments'
```

<mark style="background: #FFF3A3A6;">to create a pod (imperative command)</mark>
use :`kubectk run redis --image redis:alpine`

<mark style="background: #FFF3A3A6;">to create a service (imperative command)</mark>
use: `kubectl expose `
example: `kubectl expose pod redis --port 6379 --name redis-service --type ClusterIP`

<mark style="background: #FFF3A3A6;">to create a deployment (imperative command)</mark>
use: `kubectl create deployment`