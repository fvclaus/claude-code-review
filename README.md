# claude-code-review

Reusable GitHub Actions workflow that runs an AI-powered PR review using [Claude Code](https://claude.ai/code).

## Why this exists

Most AI review prompts are written from first principles and end up generic — they flag the same abstract categories ("error handling", "simplicity") that any LLM would flag anyway, and miss the patterns that *this* reviewer actually cares about.

This workflow uses a **data-driven prompt**: the review checklist was distilled from real PR comments left on this codebase using the [`/improve-reviewer-prompt`](#the-improve-reviewer-prompt-skill) Claude Code skill. The skill collected past review comments, categorised them into a taxonomy of what the reviewer actually flags, identified gaps in the existing prompt, and synthesised a rewritten version — grounded in concrete, checkable rules rather than vague guidance.

The result is a reviewer that catches what matters here (serialisation round-trips, non-idempotent writes, swallowed errors that make failed jobs look successful, …) rather than what a generic checklist says should matter.

## The `/improve-reviewer-prompt` skill

`/improve-reviewer-prompt` is a Claude Code skill that turns a reviewer's real PR comments into an improved review prompt. It runs in two tracks:

**Audit track (steps 1–6)** — collection and reporting:
1. Resolves the target scope (`current` repo, `all` repos in an org, or an explicit selection)
2. Fetches every PR comment in the chosen time window via the GitHub API, optionally filtered to one reviewer's login
3. Organises comments into a structured `{repo → PR → [comments]}` shape
4. Generates per-repo markdown reports with comment bodies and diff context

**Prompt-improvement track (steps 7–10)** — distillation:
5. Strips diff hunks to produce a compact corpus of comment bodies
6. Fans out to subagents to categorise comments into a taxonomy (each category gets a concrete, checkable rule and example comments)
7. Merges taxonomies and maps them against the existing prompt — producing a coverage table that shows which patterns the prompt already catches and which it misses
8. Synthesises a rewritten prompt: abstract buckets replaced with the distilled categories, a correctness-first ordering, and explicit per-category enumeration so the reviewer can't skip a dimension silently

To run it against the current repo:

```
/improve-reviewer-prompt
```

Claude will ask for the scope, time window, reviewer filter, and — if you want the rewrite — the current prompt to improve.

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
