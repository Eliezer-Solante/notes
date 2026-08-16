
| Command               | Purpose                                                           |
| --------------------- | ----------------------------------------------------------------- |
| `git clean -n`        | Show what untracked files/directories would be deleted (dry run). |
| `git clean --dry-run` | Same as `-n`; preview deletions without removing anything.        |
| `git clean -f`        | Delete untracked files.                                           |
| `git clean -fd`       | Delete untracked files and untracked directories.                 |
| `git clean -fdx`      | Delete untracked files, directories, and ignored files.           |
| `git clean -fdX`      | Delete only ignored files and directories.                        |
| `git clean -i`        | Start interactive mode and choose what to delete.                 |

### Comparison with `git reset --hard`

| Command                 | Affects Tracked Files? | Affects Untracked Files?        |
| ----------------------- | ---------------------- | ------------------------------- |
| `git reset --hard HEAD` | ✅ Yes                  | ❌ No                            |
| `git clean -fd`         | ❌ No                   | ✅ Yes                           |
| `git clean -fdx`        | ❌ No                   | ✅ Yes (including ignored files) |

### Recommended Safe Workflow

| Step | Command          | Purpose                                                                           |
| ---- | ---------------- | --------------------------------------------------------------------------------- |
| 1    | `git clean -nd`  | Preview files/directories that would be removed.                                  |
| 2    | `git clean -fd`  | Remove untracked files and directories.                                           |
| 3    | `git clean -fdx` | Use only if you also want to remove ignored files (e.g., `node_modules`, `dist`). |

⚠️ **Warning:** Files removed by `git clean` are generally not recoverable through Git because they were never tracked. Always run `git clean -n` or `git clean -nd` first.
