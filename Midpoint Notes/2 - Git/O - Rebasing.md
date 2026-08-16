
![[Pasted image 20260721130509.png]]

![[Pasted image 20260721130534.png]]

![[Pasted image 20260721131518.png]]


![[Pasted image 20260721131359.png]]

![[Pasted image 20260722145130.png]]
Here's the first half — the original chain, and what happens when you mark `c2` for editing mid-rebase. Now the result once you amend and continue:
![[Pasted image 20260722145155.png]]
The two diagrams together show the full cycle: git stops at your target commit, lets you fix it (whether via `--amend` or a `--fixup`/`reset --soft` approach), then replays everything after it in the same order — just with fresh hashes because each commit's identity is tied to its parent's.

