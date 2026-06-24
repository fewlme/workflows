# Audit Workflows

Reusable GitHub Actions workflows that run Claude-powered audits (code quality,
security, performance, accessibility, dependency health, documentation) on a
schedule and open labelled GitHub issues for findings.

Consumer repos reference these workflows through the moving **`@v1`** tag, so
fixes shipped here reach every project without editing each repo.

## What's in this repo

| Reusable workflow | Purpose | Default schedule (in callers) |
|---|---|---|
| `code-quality.yml` | Maintainability / architecture / complexity review | Tue 06:00 |
| `security-audit.yml` | SAST + dep audit + secret scan, triaged by Claude | Mon 07:00 |
| `performance.yml` | Runtime / build performance review | Wed 06:00 |
| `accessibility.yml` | Frontend a11y review (skips if no frontend) | Fri 06:00 |
| `dependency-health.yml` | Outdated / risky dependency report | 1st of month 06:00 |
| `docs.yml` | Documentation gaps (optional auto-fix PR) | Thu 06:00 |

Each audit creates issues only at or above its `severity_threshold` and labels
them by category + severity. The labels are created automatically — see
[Issue labels](#issue-labels).

## Setting up a new project

### 1. Add the auth secret (required)

Every audit calls the Claude action, which needs an OAuth token. Each repo needs
its own copy (the `fewlme` account has no org-level secret sharing).

```bash
# Generate a token (needs a Claude Pro/Max subscription). One token works for every repo.
claude setup-token            # prints an sk-ant-oat... value

# Store it in the new repo
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>
```

Optional: `SLACK_WEBHOOK` for run notifications.

```bash
gh secret set SLACK_WEBHOOK --repo <owner>/<repo>
```

> The token expires (~1 year). When it does, audits fail with
> `Either ANTHROPIC_API_KEY, CLAUDE_CODE_OAUTH_TOKEN, ... is required`.
> Re-run `claude setup-token` and update the secret in every repo.

### 2. Add the caller workflows

Create one small caller per audit under `.github/workflows/` in the new repo.
They all follow the same shape — `schedule` + `workflow_dispatch`, the right
`permissions`, and `uses: ...@v1`. Minimal example (`quality.yml`):

```yaml
name: Quality
on:
  schedule:
    - cron: '0 6 * * 2'   # Tuesday 6am
  workflow_dispatch:
permissions:
  contents: read
  issues: write
  pull-requests: read
  id-token: write
jobs:
  audit:
    uses: fewlme/workflows/.github/workflows/code-quality.yml@v1
    # Pass only the secrets the reusable workflow declares — never `secrets: inherit`.
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

Swap `code-quality.yml` for whichever audit you want. Two callers need extra
permissions:

- **`security-audit.yml`** → add `security-events: write`
- **`docs.yml`** → needs `contents: write` and `pull-requests: write` (it can
  open auto-fix PRs); enable them with the `create_pr: true` input.

The fastest way to bootstrap is to copy the six caller files from an existing
project (e.g. `bilbokidmodels`) and keep the `@v1` pins as-is.

### 3. Trigger a first run

```bash
gh workflow run quality.yml --repo <owner>/<repo>
```

## Inputs

All audits accept these (passed under `with:` in the caller):

| Input | Type | Default | Notes |
|---|---|---|---|
| `severity_threshold` | string | `high` | Min severity to report. Not used by `dependency-health` / `docs`. |
| `create_issues` | boolean | `true` | Set `false` for a dry run (summary only, no issues). |
| `claude_model` | string | `claude-sonnet-4-6` | Model used for the audit. |
| `create_pr` | boolean | `false` | **`docs.yml` only** — open a PR for mechanical fixes. |

## Issue labels

Each audit ensures its labels exist before filing issues (an idempotent
`gh label create --force` step), so a brand-new repo doesn't need any labels
pre-created. `gh issue create` rejects the whole command if any label is
missing, so this step is what keeps findings labelled.

| Audit | Labels applied |
|---|---|
| code-quality | `code-quality` + `critical`/`high`/`medium`/`low` |
| security-audit | `security` + severity |
| performance | `performance` + severity |
| accessibility | `accessibility` + severity |
| dependency-health | `dependencies` |
| docs | `documentation` + severity |

## Versioning (`@v1`)

Consumers pin the moving `v1` tag, **not** a commit SHA, so a fix here lands
everywhere on the next run. After merging a change to `master`, move the tag:

```bash
git tag -f v1 <new-sha>
git push -f origin v1
```

Existing open issues are not retro-labelled; the change takes effect on the next
scheduled or dispatched run.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `... CLAUDE_CODE_OAUTH_TOKEN ... is required` | Secret missing/empty in that repo | `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>` |
| Issues created without labels | Labels missing **and** running an old pin without the label-creation step | Move the repo to `@v1` |
| A fix isn't taking effect | Consumer pinned to an old SHA | Repin to `@v1`, or move the `v1` tag |
| Audit ran but filed nothing | No findings at `severity_threshold`, or `create_issues: false` | Lower the threshold / check the run summary |
