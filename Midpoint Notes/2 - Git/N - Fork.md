 ![[Pasted image 20260721115621.png]]

<mark style="background: #FFB8EBA6;">Steps:</mark> 
1 - Fork the main repo
2 - Clone the repo to the machine from the forked repo
3 - Run `cd <repository title>` to enter the repo
4 - Make new File/changes
5 - Commit and Push to the master branch
6 - Create Pull Request in the Git UI
7 - Wait for the author to review and approve your changes

 ## 🔑 Key Points About Forks

- **Independent copy** – A fork is separate from the original repository, so you can freely experiment without affecting the source.
    
- **Collaboration workflow** – Forking is the standard way to contribute to open-source projects: fork → make changes → submit a pull request.
    
- **Pull requests** – After editing your fork, you propose changes back to the original repo via a pull request.
    
- **Syncing forks** – You can update your fork with new changes from the original repository using `git fetch upstream` and `git merge` or `git rebase`.
    
- **Difference from clone** – A fork is hosted on your account (server-side), while a clone is a local copy on your machine.
    

## 🛠 Typical Workflow

1. **Fork the repository** on GitHub/GitLab.
    
2. **Clone your fork** locally with `git clone`.
    
3. Create a **new branch** for your changes.
    
4. Make edits, then **commit** and **push** to your fork.
    
5. Open a **pull request** to merge your changes into the original project.
    

## 📊 Fork vs Clone vs Branch

|Concept|Location|Purpose|
|---|---|---|
|**Fork**|Remote (your account)|Independent copy for collaboration|
|**Clone**|Local machine|Work offline on repo|
|**Branch**|Inside a repo|Parallel development within same repo|