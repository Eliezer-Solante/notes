
![[Pasted image 20260731095350.png]]

![[Pasted image 20260731095425.png]]

![[Pasted image 20260731095508.png]]

![[Pasted image 20260731095608.png]]

### Fix 1 - cache mounts / for redownloading packages on every build
![[Pasted image 20260731095733.png]]

![[Pasted image 20260731095822.png]]

### Fix 2 - secret mounts (e.g API Keys)
![[Pasted image 20260731095944.png]]

![[Pasted image 20260731100038.png]]

### Fix 3 - multi-platform builds
Builds the image twice for different architecture. (arm64 or amd64)
![[Pasted image 20260731100419.png]]
![[Pasted image 20260731100447.png]]
It pulls the compatible image for a specific architecture 

### Fix 4 - parallel stages
![[Pasted image 20260731100724.png]]
![[Pasted image 20260731100752.png]]



![[Pasted image 20260731100838.png]]


