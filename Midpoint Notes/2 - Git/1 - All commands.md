![[Pasted image 20260721125616.png]]
![[Pasted image 20260721125636.png]]
## 🚀 Getting Started

- **git init** – Initialize a new repository.
- **git clone** – Copy a remote repository locally.
- **git config** – Set username/email for commits.
    
## 📂 Tracking Changes

- **git status** – Show modified, staged, and untracked files.
- **git diff** – Compare changes (unstaged vs staged).
- **git add** – Stage changes for commit.
- **git commit** – Save staged changes as a snapshot.
- **git commit --amend** – Modify the last commit.
    
## 🧭 Branching & Switching

- **git branch** – List, create, or delete branches.
- **git switch** / **git checkout** – Move between branches.
- **git branch -d** – Delete merged branch.
- **git tag** – Mark commits (e.g., releases).
    
## 🔄 Merging & Rebasing

- **git merge** – Combine another branch into current.
- **git rebase** – Reapply commits for linear history. 
- **git cherry-pick** – Apply a specific commit.
- **git rebase -i** – Interactive rebase for squashing/reordering.
    
## 🌐 Remote Collaboration

- **git remote -v** – Show linked remotes.
- **git fetch** – Download changes without merging.
- **git pull** – Fetch + merge changes.
- **git push** – Upload commits to remote.
- **git push --tags** – Push tags to remote.
    
## 📜 History & Inspection

- **git log** – View commit history.
- **git log --oneline --graph** – Compact visual history.
- **git show** – Display details of a commit. 
- **git blame** – Show who changed each line.
- **git reflog** – Recover lost commits.
    

## 🛠 Undo & Cleanup

- **git restore** – Discard local changes.
- **git reset** – Undo commits (soft/hard).
- **git revert** – Create a new commit that undoes changes.
- **git clean** – Remove untracked files.
    
## 📦 Stashing

- **git stash** – Temporarily save uncommitted changes.
- **git stash pop** – Reapply and remove stash.
- **git stash list** – Show stashed changes.

---

## 📜 Basic Usage of `Git log`

- **git log** – Shows commit history with author, date, and message.
- **git log --oneline** – Compact view: one commit per line. 
- **git log --graph** – ASCII graph of branches and merges.
- **git log --decorate** – Shows branch and tag names alongside commits.
    
## 🔍 Filtering Commits

- **git log --author** – Show commits by a specific author.
- **git log --grep** – Search commit messages for keywords.   
- **git log --since** / **git log --until** – Filter commits by date. 
- **git log -p** – Show actual changes (diffs) for each commit.
    

## 🎨 Formatting Output

- **git log --stat** – Show file changes (insertions/deletions).
- **git log --pretty=oneline** – One-line format. 
- **git log --pretty=format** – Custom output (e.g., `%h %an %s`).   
- **git log --abbrev-commit** – Short commit hashes.
    

## 📂 Range & Branch-Specific Logs

- **git log branch_name** – Show commits for a specific branch.
- **git log commit1..commit2** – Show commits between two points.
- **git log --merges** – Show only merge commits.  
- **git log --no-merges** – Exclude merge commits.
    

## 🛠 Advanced Tricks

- **git log -n** – Limit number of commits shown.
- **git log --follow** – Track history of a single file across renames.
- **git log --reverse** – Show commits in chronological order.
- **git log --cherry** – Compare commits between branches.


git remote set-url --push upstream DISABLE
git remote add upstream https://github.com/<org>/Git-Practice.git