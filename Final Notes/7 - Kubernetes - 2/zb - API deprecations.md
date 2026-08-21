
![[Pasted image 20260822002130.png]]

![[Pasted image 20260822002259.png]]

![[Pasted image 20260822002320.png]]
![[Pasted image 20260822002606.png]]

![[Pasted image 20260822002643.png]]

![[Pasted image 20260822002805.png]] 

 ![[Pasted image 20260822003121.png]]

 ![[Pasted image 20260822003250.png]]![[Pasted image 20260822003317.png]]
 
 ![[Pasted image 20260822003341.png]]

 ![[Pasted image 20260822003406.png]]

 ![[Pasted image 20260822003540.png]]
`kubectl convert -f ingress-old.yaml --output-version networking.k8s.io/v1 > ingress-new.yaml`

`kubectl create -f ingress-new.yaml `
 ![[Pasted image 20260822003620.png]]

`https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-convert-plugin`

 `kube-apiserver.yaml` file is usually located in here: `/etc/kubernetes/manifests/kube-apiserver.yaml`