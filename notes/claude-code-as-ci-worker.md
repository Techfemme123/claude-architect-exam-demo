# Claude Code as a CI Worker

Study note for issue #13 (originally #4). Topic list is **unverified**: no official exam guide is in the repo, so this note follows the issue outline, not confirmed exam topics.

See also: [`github-actions-and-ci-triggers.md`](../github-actions-and-ci-triggers.md) for triggers, permissions, and debugging.

## 1. Get the `gh` CLI

- Install: <https://cli.github.com> (`brew install gh`, `winget install GitHub.cli`, or your Linux package manager).
- Log in: `gh auth login`, then check with `gh auth status`.
- Preinstalled on GitHub-hosted `ubuntu-latest` runners; in Actions it authenticates via the `GH_TOKEN` env var.
- Commands a CI worker uses most:

| Task | Command |
|------|---------|
| List runs | `gh run list` |
| Inspect a run | `gh run view <id>` |
| Failed logs only | `gh run view <id> --log-failed` |
| PR check status | `gh pr checks <n>` |
| Read an issue | `gh issue view <n>` |
| Open a PR | `gh pr create --base main` |

## 2. CI concept using a GitHub project

CI = every change is automatically built and checked before it merges.

Loop in this repo:

1. Push a branch, open a PR to `main`.
2. `pull_request` event starts workflows.
3. Checks run (here: `markdown-only`, Claude code review).
4. Red check: fix and push again. Green: merge.

Where Claude fits: it is another job on the runner, started by an event. It reads context, edits files, pushes, and reports back.

## 3. Live demo: Claude as CI worker

Demo script (uses this repo):

1. Open an issue describing a small change.
2. Comment `@claude` with the request.
3. `Claude Code` workflow starts (Actions tab).
4. Claude updates a single tracking comment with a checklist.
5. It commits to a `claude/issue-N-...` branch and posts a "Create PR" link.
6. Open the PR. Checks run. Claude review comments on it.
7. Check results: `gh pr checks <n>`.

Observe: the job's `if:` gate, the progress comment, the branch name, and the checks on the PR.

## 4. In-depth configuration of Claude Code as a CI worker

### Setup

- Install the Claude GitHub app / run `/install-github-app` in Claude Code.
- Add the secret `CLAUDE_CODE_OAUTH_TOKEN` (or an API key) under repo Settings > Secrets.
- Add workflow files under `.github/workflows/` (must be committed by a human; the app cannot push workflow changes).

### Workflow anatomy (from `claude.yml`)

```yaml
on:
  issue_comment: { types: [created] }
  issues: { types: [opened, assigned] }
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')   # filter precisely
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read          # read CI results
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 1 }
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # prompt: "..."              # optional; default = follow the comment
          # claude_args: '--allowed-tools Bash(gh pr *)'
```

### Key knobs

| Setting | Purpose |
|---------|---------|
| `prompt` | Fixed instruction (automation mode). Omit to act on the `@claude` comment. |
| `claude_args` | CLI flags: `--allowedTools`, `--max-turns`, `--model`, etc. |
| `--allowedTools` | Allow-list for tools such as `Bash(gh pr *)`. Unlisted tools are denied. |
| `additional_permissions` | Extra token scopes, e.g. `actions: read`. |
| `plugins` / `plugin_marketplaces` | Load plugins such as the code-review plugin (see `claude-code-review.yml`). |
| `CLAUDE.md` | Repo rules Claude reads first (e.g. markdown only). |

### Modes

- **Interactive** (`@claude` in comments): `claude.yml`.
- **Automatic** (every PR): `claude-code-review.yml` on `pull_request`.

### Safety checklist

- Least-privilege `permissions:`; narrow `--allowedTools`.
- Treat issue/PR text as untrusted; never interpolate it into `run:` scripts.
- Secrets only via `${{ secrets.* }}`; none on fork PRs.
- Commits pushed with `GITHUB_TOKEN` do not trigger new runs.
- Never skip or disable checks to get green.

## Practice questions

1. Why does `claude.yml` subscribe to four events but gate the job with `if:`?
2. Claude pushed a commit, but `pull_request` checks did not start. Likely cause?
3. Which permission lets Claude read CI results, and where is it set?
4. How would you restrict Claude to only `gh pr` commands?
5. Why must a human add or edit files in `.github/workflows/`?

<details>
<summary>Answers</summary>

1. Subscribe broadly, filter precisely: skipped jobs are cheap, unfiltered agent runs are not.
2. The push used `GITHUB_TOKEN`, which does not trigger new workflow runs.
3. `actions: read` in `permissions:` (and `additional_permissions` for the action).
4. `claude_args: '--allowed-tools Bash(gh pr *)'`.
5. The GitHub App lacks permission to modify workflow files.

</details>
