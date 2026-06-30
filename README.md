# claude-code-review

Reusable GitHub Actions workflow that runs an AI-powered PR review using [Claude Code](https://claude.ai/code).

## Why this exists

Without a detailed prompt, Claude misses a lot in code review. The checklist in this workflow was built using the [`/improve-reviewer-prompt`](#the-improve-reviewer-prompt-skill) Claude Code skill, which distilled real review comments I had left across several codebases — things Claude consistently missed — into concrete, checkable rules.

## The `/improve-reviewer-prompt` skill

A Claude Code skill that turns a reviewer's past PR comments into an improved review prompt. Given a repo scope and time window, it:

1. Fetches PR comments via the GitHub API (optionally filtered to one reviewer)
2. Categorises them into a taxonomy — each category gets a checkable rule and example comments
3. Maps the taxonomy against an existing prompt to find gaps
4. Synthesises a rewritten prompt with the distilled categories, correctness-first

```
/improve-reviewer-prompt
```

Claude will ask for scope, time window, reviewer filter, and the current prompt to improve.

## Usage

```yaml
# .github/workflows/claude.yml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude'))
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    uses: fvclaus/claude-code-review/.github/workflows/review.yml@main
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Trigger a review by commenting `@claude review please` on any pull request.

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `prompt` | No | Custom instructions prepended before the standard review checklist. Use this to add project-specific context (e.g. point Claude at your `AGENTS.md`). |

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `claude_code_oauth_token` | Yes | Claude Code OAuth token from [claude.ai](https://claude.ai). Store as a repository or organisation secret. |

## Required permissions

The calling job must grant:

| Permission | Level | Why |
|------------|-------|-----|
| `pull-requests` | `write` | Post the review comment |
| `issues` | `write` | Post comments on issues linked to PRs |
| `actions` | `read` | Read CI run results to include in the review |
| `id-token` | `write` | OIDC token exchange for Claude authentication |
| `contents` | `read` | Check out the repository |

A preflight step verifies these at the start of each run and emits a clear error if any are missing.

## Custom prompt example

```yaml
    uses: fvclaus/claude-code-review/.github/workflows/review.yml@main
    with:
      prompt: |
        You are reviewing a Django REST API. Read AGENTS.md in the repo root
        for architecture rules before starting.
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The custom prompt is prepended before the standard checklist, so Claude applies both your project context and the default review criteria.
