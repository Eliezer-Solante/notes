Here's the complete end-to-end walkthrough with the upstream write-guard folded in.

## 0. Initial setup (one-time)

```bash
# Clone your fork
git clone https://github.com/<your-username>/Git-Practice.git
cd Git-Practice

# Add the shared repo as "upstream"
git remote add upstream https://github.com/<org>/Git-Practice.git

# Disable push access to upstream — read-only guardrail
git remote set-url --push upstream no_push

# Confirm remotes are set up correctly
git remote -v
```

Expected output:

```
origin    https://github.com/<you>/Git-Practice.git (fetch)
origin    https://github.com/<you>/Git-Practice.git (push)
upstream  https://github.com/<org>/Git-Practice.git (fetch)
upstream  no_push (push)
```

This means: you can `fetch upstream` freely, but any accidental `git push upstream ...` will fail immediately instead of pushing to the shared repo.

---

## Challenge 1 — Fix the birthdate and open the PR

### 1. Create your branch from `feature_2508B`

```bash
git fetch upstream
git checkout -b Andres_BD_fix upstream/feature_2508B
```

Check tracking right away — branching from `upstream/...` can set your push target to `upstream`:

```bash
git status
```

If needed, point it at your fork instead:

```bash
git push -u origin Andres_BD_fix
```

(With the write-guard in place, even if tracking had defaulted to `upstream`, a push attempt would simply fail rather than silently landing on the shared repo — but it's still good practice to confirm tracking explicitly.)

### 2. Fix the birthdate

Edit `Config/Users/user_basic_info/abonifacio.json`:

- Change `1992-11-30` → `1982-09-02`
- Save.

### 3. Commit and push

```bash
git add Config/Users/user_basic_info/abonifacio.json
git commit -m "chore: Updated Andres Bonifacio Details"
git push origin Andres_BD_fix
```

### 4. Open the PR

- GitHub → New PR: `Andres_BD_fix` → **`feature_2508A`**.
- Title: `Academy Intake <Batch Number> - <Your Name>`.
- Expect unrelated files to appear (from the Exercise 5 commit on `feature_2508B`) — that's expected at this point.

### 5. Verification for Challenge 1

- **Files changed** tab → comment on any line → screenshot showing the unrelated files present.
- **Conversation** tab → paste as text:

```bash
git diff --stat upstream/feature_2508A Andres_BD_fix
```

- **Do not merge.**

---

## Challenge 2 — Clean out unrelated changes

### 1. Sync up

```bash
git checkout Andres_BD_fix
git fetch upstream
```

### 2. Soft-reset onto the PR's target branch

```bash
git reset --soft upstream/feature_2508A
```

### 3. See what's staged

```bash
git status
```

Expected (per your actual output):

```
modified:   Config/Users/user_address/abonifacio.json
modified:   Config/Users/user_address/jdelacruz.json
modified:   Config/Users/user_basic_info/abonifacio.json
modified:   Config/Users/user_preference/jdelacruz.json
modified:   Config/Users/user_preference/msantos.json
```

### 4. Restore the unrelated files to `feature_2508A`'s version

```bash
git restore --source=upstream/feature_2508A --staged --worktree Config/Users/user_address/jdelacruz.json
git restore --source=upstream/feature_2508A --staged --worktree Config/Users/user_preference/jdelacruz.json
git restore --source=upstream/feature_2508A --staged --worktree Config/Users/user_preference/msantos.json
```

### 5. Confirm only Bonifacio's files remain

```bash
git status
```

Expected:

```
modified:   Config/Users/user_address/abonifacio.json
modified:   Config/Users/user_basic_info/abonifacio.json
```

### 6. Commit

```bash
git commit -m "Removing Unnecessary changes"
```

### 7. Double-check before pushing

```bash
git log --oneline
git diff --stat upstream/feature_2508A Andres_BD_fix
git status
```

- `git log --oneline` → 1 commit on top of `feature_2508A`.
- `git diff --stat` → only the 2 abonifacio files.
- `git status` → confirm tracking is `origin`, not `upstream` (and even if it weren't, the write-guard would block it).

### 8. Force-push the rewritten branch

```bash
git push --force-with-lease origin Andres_BD_fix
```

### 9. Verification for Challenge 2

On the same PR, **Files changed** tab:

- Comment with a screenshot of `git log --oneline` (≤2 commits) or the PR's commit-count badge.
- Same or follow-up comment: screenshot of **Files changed** showing exactly:
    - `Config/Users/user_basic_info/abonifacio.json`
    - `Config/Users/user_address/abonifacio.json`

---

## Optional cleanup afterward

If you ever need push access to `upstream` restored (e.g., different task requiring it):

```bash
git remote set-url --push upstream https://github.com/<org>/Git-Practice.git
```

That's the full flow, start to finish, with the guardrail active throughout so nothing accidentally lands on the shared repo.