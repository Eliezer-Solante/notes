Happy to dig into this — Git hooks are one of those features that seem obscure until you understand the mechanics, and then they're everywhere in real workflows (CI/CD, code review gates, notifications, etc.).

## What a Git hook actually is

A Git hook is just a script that Git automatically runs at specific points in its workflow — before a commit, after a push, before a merge, and so on. There's nothing magical about them: they're plain executable scripts (Bash, Python, Perl, whatever you like) sitting in a specific folder, and Git calls them by convention, based on filename, at the right moment.

**Where they live:**

```
<repo>/.git/hooks/
```

or for a bare repo (like `/opt/cluster.git`):

```
/opt/cluster.git/hooks/
```

If you `ls` that directory in a fresh repo, you'll see a bunch of `.sample` files — Git ships example hooks for every trigger point, but they're disabled by default (the `.sample` extension means Git ignores them). To activate one, you **remove the `.sample` extension** and make the file **executable**.

## Client-side vs. server-side hooks

This is the distinction that matters most for understanding your task:

**Client-side hooks** — run on your local machine, triggered by local Git operations:

| Hook          | Fires when                                                         |
| ------------- | ------------------------------------------------------------------ |
| `pre-commit`  | Before a commit is finalized — great for linting/tests             |
| `commit-msg`  | After you write a commit message — can validate/reject formatting  |
| `post-commit` | After a commit completes — notifications, logging                  |
| `pre-push`    | Before `git push` sends anything — last chance to block a bad push |

**Server-side hooks** — run on the machine hosting the repo (typically a bare repo), triggered when someone pushes to it:

| Hook           | Fires when                                                                                                                                                                             |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pre-receive`  | Before any refs are updated — can reject the entire push                                                                                                                               |
| `update`       | Once per branch being updated — can reject individual branches                                                                                                                         |
| `post-receive` | After all refs are updated — good for deployment, notifications, CI triggers                                                                                                           |
| `post-update`  | Similar to `post-receive`, but simpler — receives just a list of ref names that were updated, commonly used for tasks like updating server info or, as in your task, tagging a release |

This is exactly why the earlier task placed the hook in `/opt/cluster.git/hooks/` rather than the working clone — the bare repo is the "server," and it's the one receiving the push.

## Why `post-update` specifically made sense there

- It runs **after** the push is fully accepted, so the new commits genuinely exist by the time it fires — appropriate for tagging a completed release.
- It's simpler than `post-receive`: it just gets a list of ref names (like `refs/heads/master`) as arguments, without the more detailed old-SHA/new-SHA info `post-receive` gets via stdin.
- If you needed to *reject* a push based on some condition (e.g., blocking direct pushes to `master`, enforcing commit message formats), you'd use `pre-receive` or `update` instead, since `post-update` can't stop a push — the changes are already committed to the repo by the time it runs.

## A few practical things worth knowing

**Hooks aren't cloned or version-controlled by default.** They live in `.git/hooks/`, which is excluded from what gets pushed/pulled — so if you clone a repo, you don't inherit its hooks. This is why teams that rely heavily on hooks often use a tool like **Husky** (for Node projects) or symlink a tracked `hooks/` directory into `.git/hooks/` via a setup script, so hooks travel with the codebase.

**They must be executable.** This trips people up constantly:

```bash
chmod +x .git/hooks/post-update
```

Without this, Git silently ignores the hook — no error, it just doesn't run.

**They can be written in any language**, as long as the file has a proper shebang line (`#!/bin/bash`, `#!/usr/bin/env python3`, etc.) and is executable. Git doesn't care what's inside, just that it's runnable.

**Exit codes matter for "blocking" hooks.** For hooks like `pre-commit`, `pre-push`, or `pre-receive`, returning a non-zero exit code aborts the operation. `post-*` hooks run *after* the fact, so their exit code doesn't block anything — they're for side effects (tagging, notifications, deployments), not gatekeeping.

## Common real-world use cases

- `pre-commit` → run linters/formatters, block commits with debug statements left in code
- `commit-msg` → enforce conventional commit format (e.g., `feat: `, `fix: `)
- `pre-push` → run the test suite before allowing a push
- `post-receive` → trigger a deployment script, notify a Slack channel, kick off CI

Let me know if you want to walk through writing a different hook — like a `pre-commit` that blocks commits containing a certain string, which is a very common ask.