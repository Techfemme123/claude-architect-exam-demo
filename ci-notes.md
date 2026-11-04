# Techfemme's CI Notes for Exam Prep

Notes on using Claude Code as a CI worker with GitHub.

## 1. Get the `gh` CLI

- Install: https://cli.github.com (macOS: `brew install gh`; Debian/Ubuntu: `sudo apt install gh`; Windows: `winget install GitHub.cli`).
- Authenticate: `gh auth login`, then check with `gh auth status`.
- Commands worth knowing:
  - `gh issue list` / `gh issue view <n>` / `gh issue create`
  - `gh pr create` / `gh pr view` / `gh pr checks`
  - `gh run list` / `gh run view <id> --log-failed`
  - `gh api <endpoint>` for anything else
- On GitHub Actions runners `gh` is preinstalled and uses `GITHUB_TOKEN`.

## 2. CI concept using a GitHub project

CI means every change is built, tested and reviewed automatically before merge.

1. Create a repo and keep notes or code on a branch, never directly on `main`.
2. Open an issue describing the work (this file came from issue #8).
3. Open a pull request. Workflows in `.github/workflows/` run on PR events.
4. Checks (tests, lint, review) report back on the PR.
5. Merge only when checks pass.

Optional: use a GitHub Project board to track issues through Todo, In Progress and Done.

Workflows in this repo:

| File | Purpose |
|------|---------|
| `claude.yml` | Runs Claude when `@claude` is mentioned in issues, PRs or comments |
| `claude-code-review.yml` | Automated Claude review of pull requests |
| `md-only.yml` | Check for markdown-only changes |

## 3. Live demo: Claude as a CI worker

Demo flow:

1. Create an issue describing a task.
2. Comment `@claude Work on this issue and create a PR to main.`
3. The `claude.yml` workflow starts. Claude reads the issue, creates a branch (`claude/issue-<n>-...`), makes the change, commits and pushes.
4. Claude replies in a single tracking comment with a checklist and a link to create the PR.
5. A human reviews and merges the PR.

## 4. In-depth configuration of Claude Code as a CI worker

Uses the `anthropics/claude-code-action@v1` action.

**Triggers** (`on:`): `issue_comment`, `pull_request_review_comment`, `issues`, `pull_request_review`. An `if:` condition filters on `@claude` in the body.

**Authentication**: store `CLAUDE_CODE_OAUTH_TOKEN` (or `ANTHROPIC_API_KEY`) as a repository secret and pass it via `with:`. Never commit it.

**Permissions**: grant the least needed, e.g. `contents: read`, `pull-requests: read`, `issues: read`, `id-token: write`, and `actions: read` so Claude can read CI results.

**Key inputs**
- `prompt`: fixed instructions. If omitted, Claude follows the comment that mentioned it.
- `claude_args`: CLI flags, e.g. `--allowed-tools "Bash(gh pr *)"`, `--max-turns`, `--model`.
- `additional_permissions`: extra permissions such as `actions: read`.

**Project guidance**: add a `CLAUDE.md` at the repo root with conventions, build and test commands. Claude reads it on every run.

**Safety tips**
- Restrict tools with `--allowed-tools`.
- Keep humans in the loop: Claude opens PRs, people merge.
- Claude cannot modify files in `.github/workflows/` with the default app permissions.
- Use branch protection and required checks on `main`.
