# CI Notes (Prep 2): Claude Code as a CI Worker

## 1. Get the `gh` CLI

The GitHub CLI (`gh`) lets you work with issues, PRs and Actions from the terminal, and is what Claude uses to talk to GitHub.

```bash
# macOS
brew install gh
# Debian/Ubuntu
sudo apt install gh
# Windows
winget install --id GitHub.cli

gh auth login          # authenticate (browser or token)
gh auth status         # verify
```

Useful commands:

```bash
gh issue list / gh issue view <n>
gh pr create --base main --fill
gh pr checks <n>
gh run list / gh run watch / gh run view --log-failed
```

> `gh` is preinstalled on GitHub-hosted runners; in Actions it authenticates via `GH_TOKEN: ${{ github.token }}`.

## 2. CI Concept Using a GitHub Project

Core idea: treat the repo's issues/PRs as the work queue and Claude as the worker that picks items up.

1. **Issue** describes the task (this file came from issue #6).
2. A trigger (e.g. `@claude` mention, label, or assignment) starts a workflow.
3. Claude checks out a branch, makes changes, commits and pushes.
4. Claude opens a **PR** to `main`.
5. Other CI checks (e.g. `markdown-only.yml`) run on the PR; an automated review (`claude-code-review.yml`) can comment.
6. A human reviews and merges.

Optionally use a GitHub Project board (Todo → In Progress → In Review → Done) with built-in automations to move cards as issues/PRs change state.

Key concepts to know for the exam:
- Triggers: `issue_comment`, `issues`, `pull_request`, `pull_request_review_comment`, `workflow_dispatch`, `schedule`.
- Keep a human in the loop: Claude proposes via PRs, humans merge.
- Least privilege: grant only needed `permissions:` to the workflow.

## 3. Live Demo: Claude as a CI Worker

Walkthrough to reproduce:

1. Create an issue describing a small task.
2. Comment `@claude work on this issue and create a pr to main`.
3. Watch the run in the **Actions** tab (`gh run watch`).
4. Claude posts a progress checklist in a single comment, creates branch `claude/issue-<n>-<timestamp>`, commits and pushes.
5. Open the PR link Claude provides, check CI results, review, merge.

What to observe: the tracking comment, branch naming, commit co-author trailer, and the job run link.

## 4. In-Depth Configuration of Claude Code as a CI Worker

### Setup
- Install the Claude GitHub app and add the `ANTHROPIC_API_KEY` (or OAuth token) as a repository secret. `/install-github-app` in Claude Code does this interactively.
- Workflow file: `.github/workflows/claude.yml` using `anthropics/claude-code-action@v1`.

### Example workflow

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]

jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Configuration levers
| Option | Purpose |
|---|---|
| `prompt` | Fixed instructions for automation (vs. interactive `@claude`) |
| `claude_args` | CLI flags, e.g. `--max-turns`, `--model`, `--allowedTools` |
| `--allowedTools` / `--disallowedTools` | Restrict which tools/commands Claude may run |
| `trigger_phrase` | Change the `@claude` mention |
| `CLAUDE.md` | Repo-level guidelines Claude reads every run |
| `permissions:` block | Scope of the GITHUB_TOKEN |
| Secrets | Never hard-code keys; use `${{ secrets.* }}` |

### Best practices
- Write a `CLAUDE.md` with build/test commands and conventions.
- Limit tools (`--allowedTools`) and `--max-turns` to control risk and cost.
- Use branch protection and required reviews so Claude can't merge unreviewed.
- Note: the action's GitHub App cannot modify `.github/workflows/` files.
- Use concurrency groups and timeouts to avoid runaway jobs.
- Automated review: run Claude on `pull_request` (see `claude-code-review.yml`).
- Headless use outside the action: `claude -p "<prompt>" --output-format json`.

## Checklist
- [ ] Install and authenticate `gh`
- [ ] Understand issue → branch → PR → review flow
- [ ] Run the live demo
- [ ] Know the key workflow configuration options
