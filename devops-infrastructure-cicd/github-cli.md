---
layout: default
title: GitHub CLI
parent: Infrastructure as Code
nav_order: 5
---

## [GitHub CLI](https://cli.github.com/manual/) (`gh pr`)

**Flow:** Branch → commit → push → `gh pr create` → reviewer approves → `gh pr merge` - all without leaving the terminal.

Switching to the GitHub UI to create a PR, copy a link, assign reviewers, and merge breaks flow. `gh` keeps you in the terminal where your context already lives.

### Common Workflow

```bash
# Create a PR with title, body, and reviewer
gh pr create \
    --title "Add distributed lock to invoice generation" \
    --body "Prevents duplicate invoices when SQS delivers the same message to multiple pods." \
    --reviewer teammate-handle \
    --label "backend"

# List open PRs assigned to you
gh pr list --assignee @me

# View a specific PR (diff, checks, comments)
gh pr view 142
gh pr diff 142

# Check CI status
gh pr checks 142

# Approve and merge (squash)
gh pr review 142 --approve --body "LGTM"
gh pr merge 142 --squash --delete-branch

# Quick: create PR from current branch with editor
gh pr create --fill  # auto-fills title from branch, body from commits
```

### Checking Out a Teammate's PR Locally

```bash
gh pr checkout 142
# runs tests locally, reviews code, then:
gh pr review 142 --approve
```

### Linking PRs to Issues

```bash
gh pr create \
    --title "Fix race condition in vehicle upsert" \
    --body "Closes #87. Uses ON CONFLICT instead of check-then-insert."
```

GitHub auto-closes issue `#87` when the PR merges.

> **Reliability Note:** `gh pr merge --auto` enables auto-merge when all checks pass - useful for dependabot PRs. But it merges the moment checks go green, even if you pushed a "wait, one more fix" commit that hasn't been CI'd yet. Use `--auto` only for PRs where the branch is final. For active development branches, merge manually after confirming the latest commit passed.
