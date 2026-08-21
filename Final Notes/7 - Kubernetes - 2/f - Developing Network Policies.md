Ingress
![[Pasted image 20260820104345.png]]
we only want the `api pod` in the `prod` namespace to connect to the `db` pod in the `prod` namespace, not the `api pods` in different namespaces with the same label. So add `namespaceSelector:`

what if you want to allow a `backup server` that is outside the cluster to connect to the `db` pod
![[Pasted image 20260820105013.png]]
use `ipBlock:` `from` definition 

Egress
What if the `DB` pod has an agent to send backup to the `backup server`, then allow `egress` because traffic is originating from the `DB` pod.
![[Pasted image 20260820105635.png]]