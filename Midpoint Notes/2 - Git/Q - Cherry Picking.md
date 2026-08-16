

![[Pasted image 20260721140546.png]]
**In Git,** _**cherry-picking**_ **means selecting a specific commit from one branch and applying it directly onto another branch. Unlike merging, which brings in all changes, cherry-pick lets you copy just the commit(s) you want.**

## 🔑 What Cherry-Picking Does

- **Selective commit transfer** – You choose one or more commits from another branch and replay them on your current branch.
    
- **New commit created** – Even though the changes are identical, Git generates a new commit with a different hash.
    
- **Conflict handling** – If the changes don’t apply cleanly, Git stops and lets you resolve conflicts manually.
    
- **Useful for hotfixes** – Quickly apply bug fixes from a feature branch to `main` without merging unrelated work.
    

## 🛠 Syntax

bash

```
git cherry-pick <commit-hash>
```

- **Single commit**: `git cherry-pick a3f1c92`
    
- **Multiple commits**: `git cherry-pick a3f1c92 d8b22e1`
    
- **Range of commits**: `git cherry-pick a3f1c92^..7c4e003`
    

## 📂 Typical Use Cases

- **Bug fixes** – Apply a patch commit from a feature branch directly to production.
    
- **Undo mistakes** – Move a commit accidentally made on the wrong branch to the correct one.
    
- **Backporting** – Copy a fix from the latest branch to an older maintenance branch.
    
- **Feature sharing** – Reuse a commit across multiple branches without merging everything.

## ⚠️ Risks & Limitations

- **Duplicate commits** – Cherry-picking can create duplicates if the same commit later gets merged.
    
- **Not a best practice for large changes** – Use **merge** or **rebase** when you need full branch integration.
    
- **Conflict resolution required** – Just like merging, cherry-picking may require manual fixes.
    

## 📊 Comparison: Cherry-Pick vs Merge vs Rebase

|Command|Purpose|Pros|Cons|
|---|---|---|---|
|**Cherry-pick**|Apply specific commits|Precise, flexible|Risk of duplicates|
|**Merge**|Combine branches|Preserves history|Brings all changes|
|**Rebase**|Replay commits|Clean linear history|Can rewrite history|