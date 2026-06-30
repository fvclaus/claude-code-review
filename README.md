# claude-code-review

Reusable GitHub Actions workflow that runs an AI-powered PR review using [Claude Code](https://claude.ai/code).

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
