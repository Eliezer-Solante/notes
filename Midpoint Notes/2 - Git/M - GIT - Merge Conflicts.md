![[Pasted image 20260721111934.png]] 

When you try to push a file and someone already pushed the same file, then merge conflict occurs

<mark style="background: #FFB8EBA6;">First step:</mark>
Check who pushed the latest file or the same file you were about to push
To See what was the last commit and who added a file
```
git log origin/master
```
<mark style="background: #FFB8EBA6;">Second</mark> is to pull from the latest master branch

<mark style="background: #FFB8EBA6;">Third:</mark> merging conflict will display and will download the file with the version from the master and your local version in the file content. 

![[Pasted image 20260721114146.png]]

<mark style="background: #FFB8EBA6;">Fourth</mark>: Edit the file contend using `vi` editor or any editor, and put the correct content (remove the >>>, === , HEAD, the differences. Just the correct content) and save.

<mark style="background: #FFB8EBA6;">Fifth</mark>: Commit to the local branch

<mark style="background: #FFB8EBA6;">Sixth</mark>: Push to the remote repo master

