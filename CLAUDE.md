# CLAUDE.md

Guidance for Claude when working in this repo.

## Project

Study notes for the Claude Architect exam. See `prep-plan.md` for the plan:

1. Collect the official exam guide and list its topics.
2. Write one markdown note per topic.
3. Add practice questions per topic.
4. Review weak areas and repeat.

## Rules

- **Markdown only.** The `markdown-only` CI check fails on any file that is not `.md`, `.gitignore`, `LICENSE`, or a `.github/workflows/*.yml`. Do not add other file types.
- Do not edit anything in `.github/workflows/`. The GitHub App cannot push workflow changes.
- Keep notes short and scannable: headings, bullets, small examples.
- Do not invent exam topics. If the official exam guide is not in the repo, say so and mark topic lists as "unverified".

## Layout

- `README.md`: repo overview
- `prep-plan.md`: study plan
- `notes/`: one note per topic (create when needed)

## Workflow

- Branch off `main`; `main` is the PR target.
- Use short, imperative commit messages (e.g. "Add prep plan outline").
- Open a PR for each change. Do not push directly to `main`.

## GitHub

- Tag `@claude` in an issue or PR comment to trigger the `Claude Code` workflow.
- Claude pushes to a `claude/issue-N-...` branch and leaves a "Create PR" link; open the PR from that link.
