# Audit Workflows

Reusable GitHub Actions workflows that run Claude-powered audits (code quality,
security, performance, accessibility, dependency health, documentation) on a
schedule and open labelled GitHub issues for findings — plus a forward-looking
R&D ideation workflow that proposes what to build next.

Consumer repos reference these workflows by a **full commit SHA** (with `# v1` as a
readable comment), and a per-repo **Dependabot** keeps that SHA current through
reviewed PRs. This puts a human review gate in front of any new workflow code before
it runs in your repo with write access — see [Versioning](#versioning).

## What's in this repo

| Reusable workflow | Purpose | Default schedule (in callers) |
|---|---|---|
| `code-quality.yml` | Maintainability / architecture / complexity review | Tue 06:00 |
| `security-audit.yml` | SAST + dep audit + secret scan, triaged by Claude | Mon 07:00 |
| `performance.yml` | Runtime / build performance review | Wed 06:00 |
| `accessibility.yml` | Frontend a11y review (skips if no frontend) | Fri 06:00 |
| `dependency-health.yml` | Outdated / risky dependency report | 1st of month 06:00 |
| `docs.yml` | Documentation gaps (optional auto-fix PR) | Thu 06:00 |
| `legal-compliance.yml` | Privacy/GDPR, legal-doc presence, license & content compliance | Quarterly (1st 06:00) |
| `rd-ideas.yml` | Forward-looking feature/R&D proposals (one curated report issue) | 15th of month 06:00 |
| `ui-ux.yml` | UI/UX standards, design consistency & UI-library utilization (skips if no frontend) | 8th of month 06:00 |

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
`permissions`, and a **SHA-pinned** `uses:` (see [Versioning](#versioning) for how to
get the SHA). Minimal example (`quality.yml`):

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
  id-token: write        # required: claude-code-action mints its GitHub token via OIDC
jobs:
  audit:
    uses: fewlme/workflows/.github/workflows/code-quality.yml@<40-char-sha>  # v1
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
- **`legal-compliance.yml`** → no extra permissions (issue-only, like
  `code-quality.yml`). Legal surfaces churn slowly, so a **quarterly** schedule
  keeps noise low — but the cadence is caller-controlled, set it to whatever fits:

  ```yaml
  name: Legal
  on:
    schedule:
      - cron: '0 6 1 */3 *'   # quarterly, 1st of the month 6am
    workflow_dispatch:
  permissions:
    contents: read
    issues: write
    pull-requests: read
    id-token: write        # required: claude-code-action mints its GitHub token via OIDC
  jobs:
    audit:
      uses: fewlme/workflows/.github/workflows/legal-compliance.yml@<40-char-sha>  # v1
      secrets:
        CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  ```

  Note: findings are a screening aid, not legal advice — every issue carries a
  "confirm with counsel" disclaimer.

- **`ui-ux.yml`** → no extra permissions (issue-only, like `code-quality.yml`).
  Audits design consistency, UX heuristics, and whether the project makes the
  most of the UI library it already uses (it fetches the installed version's
  docs to judge this). Skips itself when no frontend framework, UI/CSS library,
  or Blade/Twig template layer is detected. Suggested cadence: monthly on the
  8th (`cron: '0 6 8 * *'`).

- **`rd-ideas.yml`** → no extra permissions (issue-only, like `code-quality.yml`).
  It scouts the codebase and the web for new-feature opportunities and files ONE
  curated `[R&D] Ideas report` issue per run (idempotent within 30 days). A
  monthly cadence works well, offset from `dependency-health.yml` on the 1st:

  ```yaml
  name: R&D Ideas
  on:
    schedule:
      - cron: '0 6 15 * *'   # monthly, 15th 6am
    workflow_dispatch:
  permissions:
    contents: read
    issues: write
    pull-requests: read
    id-token: write        # required: claude-code-action mints its GitHub token via OIDC
  jobs:
    audit:
      uses: fewlme/workflows/.github/workflows/rd-ideas.yml@<40-char-sha>  # v1
      secrets:
        CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  ```

The fastest way to bootstrap is to copy the caller files from an existing
project (e.g. `bilbokidmodels`) and keep the SHA pins as-is — Dependabot (next step)
bumps them for you.

### 3. Add Dependabot (required)

The pins are only safe if something keeps them fresh. Add
`.github/dependabot.yml` to the new repo so the `github-actions` ecosystem is
watched — Dependabot then opens a reviewed PR whenever the pinned SHA falls behind
`v1`:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### 4. Trigger a first run

```bash
gh workflow run quality.yml --repo <owner>/<repo>
```

## Inputs

All audits accept these (passed under `with:` in the caller):

| Input | Type | Default | Notes |
|---|---|---|---|
| `severity_threshold` | string | `high` | Min severity to report. Not used by `dependency-health` / `docs` / `rd-ideas`. |
| `create_issues` | boolean | `true` | Set `false` for a dry run (summary only, no issues). |
| `claude_model` | string | `claude-sonnet-5` | Model used for the audit. |
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
| legal-compliance | `legal-compliance` + severity |
| rd-ideas | `idea` |
| ui-ux | `ui-ux` + severity |

## Versioning

Consumers pin a **full 40-char commit SHA**, not the moving `v1` tag, so a
compromised or accidental push to this repo cannot execute in every consumer on the
next run (CWE-829). The `v1` tag still exists as a human-readable "latest" marker and
as the target Dependabot tracks — each caller writes `...@<sha>  # v1`, and Dependabot
bumps the SHA toward `v1` via a **reviewed PR**. That PR is the security gate.

**Get the SHA to pin** (the commit `v1` currently points to):

```bash
gh api repos/fewlme/workflows/commits/v1 --jq .sha
```

**Maintainers — after merging a change to `master`, move the tag** so Dependabot
picks it up:

```bash
git tag -f v1 <new-sha>
git push -f origin v1
```

Consumers do **not** update instantly — they update when their Dependabot PR merges.
Existing open issues are not retro-labelled; changes take effect on the next
scheduled or dispatched run after a consumer repins.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `... CLAUDE_CODE_OAUTH_TOKEN ... is required` | Secret missing/empty in that repo | `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>` |
| Issues created without labels | Labels missing **and** running an old pin without the label-creation step | Merge the Dependabot PR (or manually repin) to the SHA `v1` points to |
| A fix isn't taking effect | Consumer pinned to an old SHA | Merge the pending Dependabot bump, or repin to the SHA from `gh api repos/fewlme/workflows/commits/v1 --jq .sha` |
| Audit ran but filed nothing | No findings at `severity_threshold`, or `create_issues: false` | Lower the threshold / check the run summary |
