# GitHub Actions and CI Triggers: A Guide for Claude and Other Agents

This guide explains how GitHub Actions work, how workflows get triggered, and what an AI agent (such as Claude) needs to know when it runs inside CI or when it triggers CI. Examples come from the workflows in this repo.

## 1. Core concepts

| Term | Meaning |
|------|---------|
| **Workflow** | A YAML file in `.github/workflows/` that defines automation. |
| **Event** | Something that happens in GitHub (push, PR opened, comment created) that can start a workflow. |
| **Job** | A group of steps that run on one runner. Jobs run in parallel unless `needs:` orders them. |
| **Step** | A single shell command (`run:`) or reusable action (`uses:`). |
| **Runner** | The VM that executes a job (e.g. `ubuntu-latest`). Fresh and ephemeral for every run. |
| **Action** | A reusable unit, e.g. `actions/checkout@v4` or `anthropics/claude-code-action@v1`. |
| **Secret** | An encrypted value (e.g. `CLAUDE_CODE_OAUTH_TOKEN`) exposed as `${{ secrets.NAME }}`. |

Minimal workflow:

```yaml
name: example
on: [push]
jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "hello"
```

## 2. CI triggers (the `on:` key)

### Common events

| Event | Fires when | Notes |
|-------|-----------|-------|
| `push` | Commits are pushed to a branch or tag | Filter with `branches:`, `tags:`, `paths:` |
| `pull_request` | A PR is opened, updated, or reopened (default types: `opened`, `synchronize`, `reopened`) | Runs on the merge commit; secrets are **not** available to PRs from forks |
| `issues` | An issue is opened, edited, assigned, labeled, etc. | Use `types:` to narrow |
| `issue_comment` | A comment is created on an **issue or a PR** | Check `github.event.issue.pull_request` to tell them apart |
| `pull_request_review` | A review is submitted | |
| `pull_request_review_comment` | An inline diff comment is created | |
| `workflow_dispatch` | Manual run from UI/API/CLI | Supports `inputs:` |
| `schedule` | Cron (UTC) | e.g. `cron: "0 6 * * 1"` |
| `workflow_call` | Another workflow calls this one | Reusable workflows |

### Filtering

```yaml
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
    paths:
      - "src/**"
```

- `types:` limits to specific activity types.
- `branches:` / `paths:` limit by target branch or changed files (`-ignore` variants exclude).
- Job-level `if:` gives finer control by inspecting the event payload.

### Important behaviors

- Workflows run from the version of the file on the branch/commit that triggered the event. For `issue_comment`, `issues`, and `schedule`, that is the **default branch**.
- Pushes made with the default `GITHUB_TOKEN` **do not trigger new workflow runs** (this prevents infinite loops). An agent that pushes a commit will not start `push`/`pull_request` workflows with that token alone. A PAT or GitHub App token is needed to chain runs.
- Pull requests from forks get a read-only token and no secrets.

## 3. How this repo uses triggers

### `claude.yml`: interactive agent ("@claude")

```yaml
on:
  issue_comment:        { types: [created] }
  pull_request_review_comment: { types: [created] }
  issues:               { types: [opened, assigned] }
  pull_request_review:  { types: [submitted] }

jobs:
  claude:
    if: contains(github.event.comment.body, '@claude') # plus per-event variants
```

Any of several events starts the workflow, then the `if:` gate only lets it proceed when the text contains `@claude`. This is the standard pattern: **subscribe broadly, filter precisely**. Skipped jobs are cheap, but unfiltered agent runs are not.

### `claude-code-review.yml`: automatic PR review

Triggered by `pull_request` (`opened`, `synchronize`, `ready_for_review`, `reopened`), so every push to a PR gets a review. The commented-out `paths:` filter shows how to limit it to relevant files.

### `md-only.yml`: policy check

Runs on `push` and `pull_request` and fails if any tracked file is not markdown (or an allowed exception). Note: a PR that adds non-markdown files will fail this check.

## 4. Permissions and security

- Declare least-privilege `permissions:` per job. This repo's agent job uses `contents: read`, `pull-requests: read`, `issues: read`, `actions: read` (to read CI results), and `id-token: write` (OIDC auth for the action).
- Note that `contents: read` means the agent's own commits are pushed through the action's app token flow, not the job's `GITHUB_TOKEN`. Workflow files under `.github/workflows/` generally cannot be modified by the app.
- Never echo secrets. Reference them only via `${{ secrets.* }}`.
- Treat issue/PR/comment text as **untrusted input**. Do not interpolate it directly into `run:` scripts (`${{ github.event.comment.body }}` in a shell is a script-injection vector). Pass it via `env:` instead.
- Pin third-party actions to a version tag or, for stricter security, a commit SHA.

## 5. Guidance for agents

**When an agent runs inside a workflow**

1. Read the repo's `CLAUDE.md` first; it holds repo-specific rules.
2. Identify the event: the trigger context (issue vs PR, comment vs review) determines which branch to use and where to reply.
3. Remember the runner is a fresh checkout (`fetch-depth: 1`): install dependencies and build before running tests, and fetch more history only if needed.
4. Only act on the instruction in the triggering comment, not other comments.
5. Report back through the designated channel (here, a single updated GitHub comment), since console output is not visible to the user.
6. If a needed tool is not allowed, say so rather than guessing; the allow-list is configured via `claude_args: --allowedTools ...`.

**When an agent's actions should trigger CI**

- Opening a PR from the agent's branch triggers `pull_request` workflows (as long as the PR is created by a user or non-`GITHUB_TOKEN` credential; otherwise they may not run).
- Check results after pushing: `gh pr checks <n>` or `gh run list/view` (requires `actions: read`).
- If a check fails, read the logs (`gh run view <id> --log-failed`), fix, push, and re-check. Do not disable or skip checks to get green.

## 6. Debugging checklist

| Symptom | Likely cause |
|---------|--------------|
| Workflow never ran | Event/`types:` mismatch, `paths:` filter excluded it, workflow file not on default branch (for issue/comment events), or the commit was pushed with `GITHUB_TOKEN` |
| Job shows "skipped" | The `if:` condition evaluated false (e.g. no `@claude` in text) |
| Secret is empty | PR from a fork, or secret not defined at repo/org/environment level |
| `Resource not accessible by integration` | Missing entry in `permissions:` |
| Duplicate runs | Both `push` and `pull_request` configured (as in `md-only.yml`); expected on PR branches in the same repo |

## 7. Quick reference

```yaml
# Run on PRs to main touching docs, plus manual runs and weekly cron
on:
  pull_request:
    branches: [main]
    paths: ["**.md"]
  workflow_dispatch:
  schedule:
    - cron: "0 6 * * 1"

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Further reading: <https://docs.github.com/actions> and <https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md>.
