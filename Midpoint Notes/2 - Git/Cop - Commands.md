Below is a beginner-friendly walkthrough of **Git commands**, with examples and simple visualizations. Git is a distributed version control system used to track changes, manage history, branch work, and collaborate with remote repositories like GitHub/GitLab/Bitbucket. [[git-scm.com]](https://git-scm.com/docs/git), [[git-scm.com]](https://git-scm.com/docs)

---

# 1. Big Picture: How Git Thinks

Git mainly works with these areas:

```text
Working Directory        Staging Area          Local Repository         Remote Repository
your files now     ->    selected changes  ->  committed history   ->   GitHub/GitLab/etc.
```

Think of it like this:

```text
Edit files
   |
   v
git add
   |
   v
Stage changes
   |
   v
git commit
   |
   v
Save snapshot locally
   |
   v
git push
   |
   v
Upload to remote repository
```

Common Git commands include `init`, `clone`, `add`, `status`, `diff`, `commit`, `branch`, `switch`, `merge`, `pull`, and `push`. [[git-scm.com]](https://git-scm.com/docs)

---

# 2. Setup Commands

Before using Git, configure your identity. This name and email appear in your commits. [[geeksforgeeks.org]](https://www.geeksforgeeks.org/git/git-cheat-sheet/)

```bash
git config --global user.name "Eliezer Solante"
git config --global user.email "eliezer@example.com"
```

Check your configuration:

```bash
git config --list
```

Check Git version:

```bash
git --version
```

---

# 3. Creating or Getting a Repository

## A. Start a New Git Project

```bash
mkdir my-project
cd my-project
git init
```

Visualization:

```text
my-project/
   |
   +-- .git/     hidden Git folder
   +-- files...
```

`git init` initializes a new Git repository in the current directory. [[geeksforgeeks.org]](https://www.geeksforgeeks.org/git/git-cheat-sheet/)

---

## B. Clone an Existing Repository

```bash
git clone https://github.com/example/project.git
```

This downloads a remote repository to your computer. `clone` is one of the standard Git commands for getting and creating projects. [[git-scm.com]](https://git-scm.com/docs)

---

# 4. Daily Git Workflow

Most of the time, you will use this flow:

```text
1. Edit files
2. Check status
3. Stage changes
4. Commit changes
5. Push changes
```

## Example

```bash
echo "Hello Git" > hello.txt
git status
git add hello.txt
git commit -m "Add hello file"
git push
```

Visualization:

```text
hello.txt modified
      |
      v
git add hello.txt
      |
      v
hello.txt staged
      |
      v
git commit -m "Add hello file"
      |
      v
snapshot saved in local Git history
```

---

# 5. Checking Status

```bash
git status
```

This shows the current state of your repository, including modified files, staged files, untracked files, and branch information. [[geeksforgeeks.org]](https://www.geeksforgeeks.org/git/git-cheat-sheet/)

Example output:

```text
On branch main
Changes not staged for commit:
  modified: index.html

Untracked files:
  style.css
```

Meaning:

```text
index.html  -> Git knows this file, but changes are not staged
style.css   -> New file Git is not tracking yet
```

---

# 6. Staging Changes

Stage one file:

```bash
git add index.html
```

Stage all changes:

```bash
git add .
```

`git add` adds files to the staging area before committing. [[geeksforgeeks.org]](https://www.geeksforgeeks.org/git/git-cheat-sheet/)

Visualization:

```text
Working Directory
   index.html modified
   style.css new
          |
          | git add .
          v
Staging Area
   index.html staged
   style.css staged
```

---

# 7. Committing Changes

```bash
git commit -m "Update homepage layout"
```

A commit saves a snapshot of staged changes into the local repository history. [[geeksforgeeks.org]](https://www.geeksforgeeks.org/git/git-cheat-sheet/)

Visualization:

```text
Commit history:

A -- B -- C
          ^
          latest commit
```

Example:

```bash
git add .
git commit -m "Add login page"
```

Good commit messages:

```text
Add login page
Fix navbar alignment
Update README instructions
Remove unused CSS
```

---

# 8. Viewing Differences

See unstaged changes:

```bash
git diff
```

See staged changes:

```bash
git diff --staged
```

`git diff` shows changes between different Git states, such as working directory, staging area, or commits. [[geeksforgeeks.org]](https://www.geeksforgeeks.org/git/git-cheat-sheet/)

Example:

```diff
- <h1>Hello</h1>
+ <h1>Hello, Git!</h1>
```

---

# 9. Viewing Commit History

```bash
git log
```

Shorter version:

```bash
git log --oneline
```

Example:

```text
a1b2c3d Add login page
e4f5g6h Update README
i7j8k9l Initial commit
```

Visualization:

```text
i7j8k9l -- e4f5g6h -- a1b2c3d
Initial     README      Login page
```

---

# 10. Branching

Branches let you work on features without affecting the main code.

Create a branch:

```bash
git branch feature-login
```

Switch to it:

```bash
git switch feature-login
```

Or create and switch in one command:

```bash
git switch -c feature-login
```

Branching and merging commands include `branch`, `switch`, `checkout`, `merge`, and related tools. [[git-scm.com]](https://git-scm.com/docs)

Visualization:

```text
main:
A -- B -- C

feature-login:
          \
           D -- E
```

Meaning:

```text
A, B, C = existing main commits
D, E    = new feature-login commits
```

---

# 11. Merging Branches

Suppose you finished your feature branch.

Switch back to main:

```bash
git switch main
```

Merge the feature branch:

```bash
git merge feature-login
```

Visualization before merge:

```text
main:          A -- B -- C
                         \
feature-login:            D -- E
```

After merge:

```text
main:          A -- B -- C -------- M
                         \        /
feature-login:            D -- E
```

`git merge` combines changes from one branch into another. [[git-scm.com]](https://git-scm.com/docs)

---

# 12. Working with Remote Repositories

## Check Remote URL

```bash
git remote -v
```

Example:

```text
origin  https://github.com/user/project.git
```

## Add a Remote

```bash
git remote add origin https://github.com/user/project.git
```

## Push to Remote

```bash
git push origin main
```

`push`, `pull`, `fetch`, and `remote` are standard Git commands for sharing and updating projects with remote repositories. [[git-scm.com]](https://git-scm.com/docs)

Visualization:

```text
Local Repository                    Remote Repository
A -- B -- C        git push       -> A -- B -- C
```

---

# 13. Pulling Latest Changes

```bash
git pull origin main
```

This downloads and integrates changes from the remote branch into your local branch. `pull` is part of Git’s sharing and updating command set. [[git-scm.com]](https://git-scm.com/docs)

Visualization:

```text
Remote:  A -- B -- C -- D
Local:   A -- B -- C

git pull

Local:   A -- B -- C -- D
```

---

# 14. Fetch vs Pull

## `git fetch`

```bash
git fetch origin
```

Downloads remote changes but does not merge them yet.

## `git pull`

```bash
git pull origin main
```

Downloads and merges changes immediately.

Visualization:

```text
git fetch:
Remote changes downloaded, but your branch is not changed yet.

git pull:
Remote changes downloaded and applied to your branch.
```

Both `fetch` and `pull` are Git commands used for updating from remote repositories. [[git-scm.com]](https://git-scm.com/docs)

---

# 15. Undoing Changes

## A. Discard Unstaged File Changes

```bash
git restore index.html
```

This restores the file from the last committed version.

## B. Unstage a File

```bash
git restore --staged index.html
```

Visualization:

```text
Staging Area
   index.html staged
          |
          | git restore --staged index.html
          v
Working Directory
   index.html modified but unstaged
```

`restore` and `reset` are Git commands commonly used for changing repository state. [[git-scm.com]](https://git-scm.com/docs)

---

## C. Undo Last Commit but Keep Changes

```bash
git reset --soft HEAD~1
```

## D. Undo Last Commit and Unstage Changes

```bash
git reset HEAD~1
```

## E. Dangerous: Remove Commit and Changes

```bash
git reset --hard HEAD~1
```

Be careful with `--hard` because it can remove local changes.

---

# 16. Stashing Temporary Work

If you are working on something but need to switch branches quickly:

```bash
git stash
```

Bring changes back:

```bash
git stash pop
```

Visualization:

```text
Current changes
      |
      v
git stash
      |
      v
Temporary storage

git stash pop
      |
      v
Changes restored
```

`stash` is one of Git’s branching and merging-related commands. [[git-scm.com]](https://git-scm.com/docs)

---

# 17. Handling Merge Conflicts

A merge conflict happens when Git cannot automatically combine changes.

Example:

```bash
git merge feature-login
```

Conflict file:

```text
<<<<<<< HEAD
Welcome to main branch
=======
Welcome to login feature
>>>>>>> feature-login
```

Fix it manually:

```text
Welcome to the login page
```

Then run:

```bash
git add index.html
git commit -m "Resolve merge conflict"
```

Visualization:

```text
Both branches edited the same line
          |
          v
Git asks you to choose/fix the final version
          |
          v
Stage and commit the resolved file
```

---

# 18. Practical Example: Full Workflow

```bash
# Clone project
git clone https://github.com/example/app.git

# Go inside project
cd app

# Create feature branch
git switch -c feature-navbar

# Edit files
echo "Navbar code" > navbar.html

# Check changes
git status

# Stage changes
git add navbar.html

# Commit changes
git commit -m "Add navbar component"

# Push branch
git push origin feature-navbar
```

Visualization:

```text
main:
A -- B -- C

feature-navbar:
          \
           D

Remote:
A -- B -- C -- D
```

---

# 19. Quick Git Cheat Sheet

```bash
git init
```

Create a new Git repository.

```bash
git clone <url>
```

Download an existing repository.

```bash
git status
```

Check current file states.

```bash
git add <file>
```

Stage a file.

```bash
git add .
```

Stage all changes.

```bash
git commit -m "message"
```

Save staged changes.

```bash
git log --oneline
```

View commit history briefly.

```bash
git branch
```

List branches.

```bash
git switch <branch>
```

Switch branch.

```bash
git switch -c <branch>
```

Create and switch to a new branch.

```bash
git merge <branch>
```

Merge another branch into the current branch.

```bash
git pull origin main
```

Get latest changes from remote.

```bash
git push origin main
```

Upload local commits to remote.

```bash
git stash
```

Temporarily save unfinished work.

```bash
git restore <file>
```

Discard local changes in a file.

```bash
git reset --soft HEAD~1
```

Undo last commit but keep changes staged.

---

# 20. Recommended Learning Path

Follow this order:

```text
1. git config
2. git init / git clone
3. git status
4. git add
5. git commit
6. git log
7. git branch
8. git switch
9. git merge
10. git push / git pull
11. git stash
12. git reset / git restore
```

If you master these commands, you can already handle most beginner-to-intermediate Git workflows. Git’s official documentation also recommends starting with tutorials and everyday commands before going deeper into the full command reference. [[git-scm.com]](https://git-scm.com/docs/git), [[git-scm.com]](https://git-scm.com/docs)