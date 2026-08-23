Replace
![[Pasted image 20260823133505.png]]
```yaml
  
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: caddy
```
to apply: `kubectl apply -k /root/code/k8s/overlays/QA`
![[Pasted image 20260823133547.png]]

Add
![[Pasted image 20260823133851.png]]
![[Pasted image 20260823133825.png]]

Remove
![[Pasted image 20260823134023.png]]
![[Pasted image 20260823134051.png]]