# GitHub Actions and CI Triggers (for Claude Code or any agent)

A guide to how GitHub Actions work, what starts a workflow (a *trigger*), and how to wire an AI agent such as Claude Code into that model. Examples refer to the workflows in this repo: `.github/workflows/claude.yml`, `claude-code-review.yml` and `markdown-only.yml`. See also `ci-notes-prep2.md`.

## 1. GitHub Actions in one minute

| Term | Meaning |
|---|---|
| **Workflow** | A YAML file in `.github/workflows/` describing automation |
| **Event / trigger** | Something that happens (push, PR opened, comment created, cron tick) and starts a workflow via `on:` |
| **Job** | A group of steps that runs on one runner; jobs run in parallel unless they declare `needs:` |
| **Step** | A shell command (`run:`) or a reusable action (`uses:`) |
| **Runner** | The machine that executes a job (`ubuntu-latest`, or self-hosted) |
| **Action** | A reusable step, e.g. `actions/checkout@v4`, `anthropics/claude-code-action@v1` |
| **Secret** | Encrypted value exposed as `${{ secrets.NAME }}` |
| **`GITHUB_TOKEN`** | Short-lived token per run; its scope is set by `permissions:` |

Flow: **event → workflow matched by `on:` → `if:` filter → job on a runner → steps → result (logs, comments, commits, PRs).**

## 2. Triggers (`on:`)

### Common triggers

| Trigger | Fires when | Typical agent use |
|---|---|---|
| `push` | Commits are pushed (filter with `branches`, `paths`, `tags`) | Lint/test; checks like `markdown-only.yml` |
| `pull_request` | PR opened, updated, reopened, etc. (`types:` narrows it) | Automated code review |
| `issues` | Issue opened, edited, assigned, labeled | Agent picks up a task when assigned or labeled |
| `issue_comment` | Comment on an issue **or PR** | `@claude` mentions |
| `pull_request_review_comment` | Inline comment on a PR diff | `@claude` requests on specific lines |
| `pull_request_review` | A review is submitted | `@claude` in review bodies |
| `workflow_dispatch` | Manual run from UI/API/`gh workflow run`, with optional `inputs:` | On-demand agent tasks |
| `schedule` | Cron (UTC), e.g. `'0 6 * * 1'` | Nightly triage, weekly dependency or doc checks |
| `workflow_run` | Another workflow finishes | Agent reacts to a failed CI run |
| `repository_dispatch` | External system calls the API | Integrations, chatops |

### Narrowing a trigger

```yaml
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
    branches: [main]
    paths: ["src/**"]      # only when matching files change
  schedule:
    - cron: "0 6 * * 1"    # Mondays 06:00 UTC
  workflow_dispatch:
    inputs:
      task:
        description: "What should the agent do?"
        required: true
```

- `types:` filters by activity (e.g. `created` vs `edited`).
- `branches` / `paths` filters reduce needless (and costly) agent runs.
- `if:` at job or step level filters on event *data*, which triggers cannot do. This is how `@claude` mention detection works:

```yaml
if: |
  (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
  (github.event_name == 'issues' && contains(github.event.issue.body, '@claude'))
```

### Gotchas

- `issue_comment` fires for both issues and PRs; check `github.event.issue.pull_request` to tell them apart.
- `schedule` and `workflow_dispatch` only run from the **default branch** version of the file.
- Workflows triggered by `GITHUB_TOKEN` actions generally don't trigger further workflows (prevents loops). A PR opened with the default token may not start `pull_request` workflows.
- PRs from forks get read-only tokens and no secrets by default. Do not run an agent with write access on untrusted fork input (prompt injection risk).
- Cron can be delayed under load and is not exact.

## 3. Two trigger models for agents

1. **Interactive / mention-driven** (`claude.yml`): a human writes `@claude ...` in an issue or PR; the agent reads that comment as its instruction, works on a branch, and reports in a single tracking comment.
2. **Automation-driven** (`claude-code-review.yml`): an event such as `pull_request` fires; a fixed `prompt:` tells the agent what to do, with no human mention needed.

Mapping triggers to agent jobs:

| Goal | Trigger |
|---|---|
| "Do what I ask in this comment" | `issue_comment`, `pull_request_review_comment` |
| "Work on this issue when assigned" | `issues: [assigned]` |
| "Review every PR" | `pull_request` |
| "Daily/weekly maintenance" | `schedule` |
| "Run on demand with parameters" | `workflow_dispatch` |
| "Diagnose failed CI" | `workflow_run` (`conclusion == 'failure'`) |

## 4. Wiring in Claude Code

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 1 }
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # prompt: "Fixed instructions for automation mode"
          # claude_args: "--max-turns 10 --allowedTools Bash(gh pr *)"
```

Key points:
- The action picks mode automatically: with a `prompt:` it runs as automation; without one it follows the `@claude` comment.
- Credentials live in secrets (`ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`), never in the file.
- `claude_args` passes CLI flags (`--max-turns`, `--model`, `--allowedTools`).
- `CLAUDE.md` in the repo gives the agent standing instructions.
- The action's GitHub App cannot edit files under `.github/workflows/`.

## 5. Any other agent

The pattern is agent-agnostic. A workflow can run any agent CLI or API client as a step:

1. Choose a trigger and filter (`on:` + `if:`).
2. Check out the code.
3. Provide credentials via secrets.
4. Run the agent headlessly, e.g. `claude -p "<prompt>" --output-format json`, or your own script.
5. Have it output via `gh` (comment, branch, PR) using `GH_TOKEN: ${{ github.token }}`.

The agent needs a prompt (from the event payload or fixed), a scoped token, restricted tools, and a place to report results.

## 6. Safety and cost checklist

- **Least privilege:** set `permissions:` explicitly; start with read-only and add write only if required.
- **Human in the loop:** agent opens PRs; branch protection and required reviews gate merges.
- **Limit blast radius:** `--allowedTools`, `--max-turns`, `timeout-minutes`.
- **Avoid runaway runs:** `concurrency:` groups with `cancel-in-progress`, and `paths`/`types` filters.
- **Untrusted input:** treat issue/PR/comment text as untrusted; don't expose secrets or write access to fork PRs.
- **Restrict who can trigger:** check `github.event.comment.author_association` or actor if the repo is public.
- **Observe:** `gh run list`, `gh run watch`, `gh run view --log-failed`.

## 7. Quick reference

```bash
gh workflow list
gh workflow run claude.yml -f task="..."   # needs workflow_dispatch
gh run list --workflow claude.yml
gh run watch
gh run view <id> --log-failed
```

## Checklist
- [ ] Can name the main triggers and when each fires
- [ ] Understand `types`, `branches`, `paths` and `if:` filtering
- [ ] Can explain mention-driven vs automation-driven agent runs
- [ ] Know the permission, secret and cost guardrails
