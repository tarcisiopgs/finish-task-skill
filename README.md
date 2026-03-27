# finish-task-skill

A Claude Code skill to finish a development cycle — open PRs, add reviewers, stop the timer, and clean up.

## What it does

Once installed, this skill automates the end of a dev cycle:

- Open PRs with cross-references across one or multiple repositories
- Add assignee and reviewers
- Stop the running timer on **Everhour** or **Toggl**
- Clean up worktrees and return to a clean state

Works best when paired with [start-task-skill](https://github.com/tarcisiopgs/start-task-skill), which saves session state.

## Prerequisites

- The **`gh` CLI** installed and authenticated
- A **time tracker** integration (Everhour skill, Toggl skill) if timer stop is desired

## Usage examples

```
/finish-task
```

No arguments needed — reads state saved by `/start-task`.

Also triggers on natural language: "terminei", "pronto", "acabei", "I'm done", "abrir PR".

## Installation

```bash
claude install tarcisiopgs/finish-task-skill
```

## License

MIT
