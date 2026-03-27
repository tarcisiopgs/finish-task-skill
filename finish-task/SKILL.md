---
name: finish-task
description: >
  Finish a development cycle: open PRs with reviewers and assignee across one or
  multiple repositories, stop the time tracker, clean up worktrees, and return to
  a clean state. Use this skill whenever the user says "finish task",
  "finalizar tarefa", "terminar tarefa", "abrir PR", "fechar tarefa", "concluir tarefa",
  "done with task", or "/finish-task". Also trigger when the user says things like
  "terminei", "pronto", "acabei", "I'm done" in the context of completing development work,
  or when the user asks to open a PR after working on a task that was started with /start-task.
---

# Finish Task

Close a development cycle by opening PRs (potentially across multiple repos), adding reviewers, stopping the timer, cleaning up worktrees, and returning to a clean state.

## Prerequisites

- The **`gh` CLI** must be installed and authenticated
- A **time tracker** skill or MCP should be configured if timer stop is desired
- Works best when paired with `/start-task`, which saves session state

## Usage

```
/finish-task
```

No arguments needed — reads state saved by `/start-task`.

## Workflow

### 1. Load session state

Read `/tmp/.claude-active-task.json`. If the file doesn't exist, ask the user for the missing context:
- Which branch to PR?
- Which repos are involved?
- Which timer to stop? (everhour/toggl, or none)
- What was the task ID?

Use the session state to determine:
- **How many repos** are involved (single vs multi)
- **What mode** was used (worktree vs branch)
- **Where each repo's working directory** is (worktreePath or gitRoot)

### 2. For each repo: check, push, and PR

Iterate over every repo in the `repos` array from the session state. For each one:

#### 2a. Check for uncommitted work

Run `git status` in the repo's working directory (worktreePath if worktree mode, gitRoot if branch mode). If there are uncommitted changes, inform the user and ask if they want to commit before proceeding. Do not auto-commit — the user should confirm the commit message.

#### 2b. Push the branch

```bash
cd <working-directory>
git push -u origin <branch>
```

If the branch was already pushed, this just ensures it's up to date.

#### 2c. Analyze changes for the PR

```bash
cd <working-directory>
git log origin/main..HEAD --oneline
git diff origin/main...HEAD --stat
```

#### 2d. Open the PR

When creating the PR, include cross-references to PRs in other repos if this is a multi-repo task. This helps reviewers understand the full scope of the change.

For the **first repo**, create the PR normally:

```bash
gh pr create --title "{concise title}" --body "$(cat <<'EOF'
## Summary
{2-3 bullet points describing what changed and why}

## Related PRs
{list of PRs in other repos, if multi-repo — fill in after all PRs are created}

## Test plan
{checklist of things to verify}
EOF
)"
```

For **subsequent repos**, include links to the PRs already created:

```bash
gh pr create --title "{concise title}" --body "$(cat <<'EOF'
## Summary
{2-3 bullet points describing what changed and why}

## Related PRs
- owner/first-repo#42

## Test plan
{checklist of things to verify}
EOF
)"
```

After all PRs are created, **go back and edit the first PR** to add the related PR links:

```bash
gh pr edit <first-pr-number> --body "$(updated body with all related PR links)"
```

This way all PRs reference each other.

#### 2e. Add assignee and reviewers

After creating each PR:

1. **Assignee**: detect the current user's GitHub username via `gh api user --jq .login` and add them as assignee with `gh pr edit --add-assignee`. Do this automatically without asking.

2. **Reviewers**: list the repository collaborators so the user can choose who to add:

   ```bash
   gh api repos/{owner}/{repo}/collaborators --jq '.[].login' | grep -v "$(gh api user --jq .login)"
   ```

   Present the list as a numbered menu (excluding the current user) and ask:
   > "Quem você quer adicionar como reviewer? (números separados por vírgula, ou 'skip' para pular)"

   Example output:
   ```
   Colaboradores disponíveis:
     1. alice
     2. bob
     3. charlie

   Quem você quer adicionar como reviewer?
   ```

   Add the selected usernames via `gh pr edit --add-reviewer user1,user2`.

   Ask once and apply the same reviewers to all PRs in the task (unless the user explicitly asks for different reviewers per repo).

   If the collaborators API call fails (e.g. insufficient permissions), fall back to asking the user to type GitHub usernames manually.

   If the user has previously configured default reviewers (e.g. via memory or CLAUDE.md), use those without asking again.

### 3. Return to clean state

The cleanup depends on the mode used:

#### Worktree mode

For each repo:

1. Switch back to the main repo directory (the original gitRoot, not the worktree)
2. Ask the user if they want to clean up the worktrees now or keep them around:
   - **Clean up**: `git worktree remove <worktreePath>` for each worktree
   - **Keep**: leave them in place (the user might want to come back to them)
3. Pull latest main: `cd <gitRoot> && git pull`

#### Branch mode

For each repo:

```bash
cd <gitRoot>
git checkout main
git pull
```

### 4. Stop the timer

Use the corresponding timer skill or MCP from the session state to stop the running timer. Present the total time tracked for this session.

#### Everhour

Use the Everhour skill to stop the current timer and retrieve the duration.

#### Toggl

Use the Toggl skill or API to stop the current time entry and retrieve the duration.

**If no timer was started or the skill is unavailable**, skip this step.

### 5. Clean up session

Delete `/tmp/.claude-active-task.json` — the cycle is complete.

### 6. Show summary

```
Task finished
  ID:      TASK-123
  Timer:   Everhour (stopped) — 1h 23m tracked

  PRs:
    api    → https://github.com/owner/api/pull/42
    iam    → https://github.com/owner/iam/pull/18

  Worktrees: cleaned up (or: kept at findup/.worktrees/)
  State:     main (up to date)
```
