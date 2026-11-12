# CI Notes: Claude Code as a CI Worker

Study notes on using Claude Code in CI/CD workflows.

## 1. Get the GitHub CLI (`gh`)

`gh` is GitHub's official command-line tool. Claude Code uses it to read and write issues, PRs, and workflow runs.

```bash
# macOS
brew install gh

# Debian/Ubuntu
sudo apt install gh

# Windows
winget install --id GitHub.cli

# Authenticate
gh auth login
gh auth status
```

Useful commands:

| Command | Purpose |
|---|---|
| `gh issue list` / `gh issue view <n>` | Read issues |
| `gh pr create` / `gh pr view <n>` | Create and inspect PRs |
| `gh pr checks` | See CI status on a PR |
| `gh run list` / `gh run view <id> --log-failed` | Inspect workflow runs and failures |
| `gh api <endpoint>` | Raw REST API calls |

In GitHub Actions, `gh` is preinstalled and authenticates through `GITHUB_TOKEN`.

## 2. CI Concept Using a GitHub Project

CI (Continuous Integration) means every change is automatically built, tested, and checked when pushed.

Core pieces in a GitHub repository:

- **Workflow**: a YAML file in `.github/workflows/`.
- **Trigger (`on:`)**: events such as `push`, `pull_request`, `issue_comment`.
- **Job**: a set of steps that runs on a runner (`runs-on: ubuntu-latest`).
- **Step**: a shell command or a reusable action (`uses:`).
- **Secrets**: sensitive values in repo settings, read as `${{ secrets.NAME }}`.

Example from this repo, `markdown-only.yml`, which fails the build if any non-Markdown file is committed:

```yaml
on:
  push:
  pull_request:
jobs:
  check-files:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: # fail if a non-.md file is tracked
```

Flow: push or open PR → workflow runs → status check appears on the PR → merge only if green.

## 3. Live Demo: Claude as a CI Worker

This repo already runs the demo:

1. Open an issue describing the task (for example, this one).
2. Comment `@claude work on this issue and create a pr to main`.
3. The `claude.yml` workflow triggers and Claude Code:
   - reads the issue and repo (`CLAUDE.md` if present),
   - creates a branch (`claude/issue-<n>-...`),
   - edits files, commits, and pushes,
   - updates a single tracking comment with a checklist and a PR link.
4. Other workflows (`markdown-only.yml`, `claude-code-review.yml`) run as checks on the resulting PR.
5. A human reviews and merges.

Things to observe during the demo: the progress checklist in the comment, the branch it creates, and the Actions run log.

## 4. In-Depth Configuration of Claude Code as a CI Worker

### Setup

- Install the Claude GitHub app, or run `/install-github-app` inside Claude Code.
- Store the credential as a repo secret: `CLAUDE_CODE_OAUTH_TOKEN` (or `ANTHROPIC_API_KEY`).

### Workflow (`.github/workflows/claude.yml`)

```yaml
on:
  issue_comment: { types: [created] }
  pull_request_review_comment: { types: [created] }
  issues: { types: [opened, assigned] }
  pull_request_review: { types: [submitted] }

jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')   # trigger phrase gate
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read          # lets Claude read CI results
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 1 }
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### Key `claude-code-action` inputs

| Input | Purpose |
|---|---|
| `claude_code_oauth_token` / `anthropic_api_key` | Authentication |
| `prompt` | Fixed instructions (automation mode). Omit to follow the `@claude` comment |
| `claude_args` | CLI flags, such as `--allowed-tools`, `--max-turns`, `--model` |
| `trigger_phrase` | Change the default `@claude` |
| `additional_permissions` | Extra GitHub permissions, such as `actions: read` |

### Controlling what Claude can do

- **Allowed tools**: restrict to what is needed, for example `--allowed-tools "Bash(gh pr *),Edit,Read"`.
- **`CLAUDE.md`**: repo-level instructions (conventions, build and test commands) that Claude reads every run.
- **Least privilege**: give the job only the `permissions:` it needs. Note that the GitHub App cannot modify files under `.github/workflows/`.
- **Cost and safety**: use `--max-turns`, set job `timeout-minutes`, and use `concurrency:` to avoid duplicate runs.
- **Human in the loop**: Claude opens PRs. Humans approve and merge, and Claude cannot approve PRs.

### Common use cases

- Implement an issue and open a PR (`@claude` comment).
- Automatic PR review on `pull_request` (`claude-code-review.yml`).
- Fix failing CI by reading `gh run view --log-failed`.
- Scheduled tasks via `on: schedule` with a `prompt`.

### Exam-style takeaways

- Claude in CI is a normal Actions step. It authenticates with a secret and acts through `gh` and git.
- Scope permissions and tools tightly. Treat Claude's output as a PR to review, not an auto-merge.
- Use `prompt` for automation and the `@claude` comment for interactive use.
