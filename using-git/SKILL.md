---
name: using-git
description: Git workflow for srt-slurm. Do not push directly to origin, push to fork and file PR. Use when performing any git operations (commit, push, branch, PR).
---

# Git Workflow for srt-slurm

When performing git operations:

1. **Never push directly to origin**:
   - `origin` (ishandhanani/srt-slurm) is upstream, treat as read-only
   - `fork` (weireweire/srt-slurm) is the user's fork, push here
   ```bash
   git push fork <branch-name>
   ```

2. **Switching branches**:
   - Always check if the branch exists on remote first (`git fetch origin <branch>` or `git fetch fork <branch>`) before creating a new one
   - If it exists remotely, track it: `git checkout -b <branch> <remote>/<branch>`
   - If you accidentally created a local branch without tracking, fix with: `git reset --hard <remote>/<branch>` and `git branch -u <remote>/<branch>`

3. **Disambiguating "提交"**:
   - "提交任务" / "提交测试" / "提交xxx测试" = submit a SLURM job (`srtctl apply`)
   - "提交代码" / "commit" = git commit
   - When ambiguous, ask for clarification

4. **Creating PRs**:
   - Push to `fork` first, then create PR specifying both repo and head:
   ```bash
   gh pr create --repo ishandhanani/srt-slurm --head weireweire:<branch-name>
   ```
   - Without `--head`, gh may create the PR from the wrong fork
   - When passing `--title` or `--body` through the shell, avoid backticks in inline text. Shell command substitution will eat content like `` `config.yaml` ``. Prefer plain text, single-quoted heredocs, or `--body-file`.
   - If `gh pr edit` fails with GitHub GraphQL/project-card errors, fall back to REST:
   ```bash
   gh api repos/ishandhanani/srt-slurm/pulls/<pr-number> --method PATCH --input <json-file>
   ```
   - When the worktree is dirty, stage only the files intended for the PR and verify with `git status --short` / `git diff --cached --stat` before committing.

5. **Destructive operations**:
   - Always confirm with user before `git reset --hard`, `git push --force`, or branch deletion
   - If accidentally pushed to origin, coordinate with user to revert
