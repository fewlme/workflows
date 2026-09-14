# Audit Workflows

Reusable GitHub Actions workflows that run Claude-powered audits (code quality,
security, performance, accessibility, dependency health, documentation, legal
compliance, UI/UX, SEO, configuration drift, live-site operations) on a
schedule and open labelled GitHub issues for findings — plus a forward-looking
R&D ideation workflow that proposes what to build next. Twelve workflows in
total.

Consumer repos reference these workflows by a **full commit SHA** (with `# v1` as a
readable comment), and a per-repo **Dependabot** keeps that SHA current. The
review gate is on this repo's `master` and the deliberate move of the `v1` tag —
see [Versioning](#versioning).

## What's in this repo

| Reusable workflow | Purpose | Example schedule (in callers) |
|---|---|---|
| `code-quality.yml` | Maintainability / architecture / complexity review | Tue 06:00 |
| `security-audit.yml` | SAST + dep audit + secret scan, triaged by Claude (Opus by default) | Mon 07:00 |
| `performance.yml` | Runtime / build performance review | Wed 06:00 |
| `accessibility.yml` | Frontend a11y review (skips if no frontend) | Fri 06:00 |
| `dependency-health.yml` | Outdated / risky dependency report (one issue per calendar month) | 1st of month 06:00 |
| `docs.yml` | Documentation gaps (optional auto-fix PR) | Thu 06:00 |
| `legal-compliance.yml` | Privacy/GDPR, legal-doc presence, license & content compliance | Quarterly (3rd 06:00) |
| `rd-ideas.yml` | Forward-looking feature/R&D proposals (one curated report issue per calendar month) | 15th of month 06:00 |
| `ui-ux.yml` | UI/UX standards, design consistency & UI-library utilization (skips if no frontend) | 8th of month 06:00 |
| `seo.yml` | Technical SEO + answer-engine (AEO) readiness + an editorial copy-suggestions pass (Opus), optional live-site checks (skips if not a web project) | 22nd of month 06:00 |
| `config-drift.yml` | Drift between code, committed env examples, compose files, Dockerfiles and CI: missing/dead env keys, compose validity, hadolint, Node/PHP version skew (skips if no config surface) | 5th of month 06:00 |
| `live-site-ops.yml` | Live scan of the hosts in `site_urls`: availability/redirects, security headers vs code, TLS certificate, SPF/DMARC/CAA/DNSSEC, domain expiry (skips if no URL given) | Sun 05:37 |

The schedules above are examples — every consumer shares one Claude
subscription, so offset them per repo (see [Scheduling](#scheduling)).

Per-finding audits create issues only at or above their `severity_threshold`
and label them by category + severity; the three report workflows
(`dependency-health`, `rd-ideas`, the `seo` editorial pass) file one report
issue per calendar month. The labels are created automatically — see
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

> The token expires (~1 year). When it does, the Claude step fails a few
> seconds after it starts with a `401` / an `is_error` result — in **every**
> repo on the same day (the Slack message says so). Re-run `claude setup-token`
> and update the secret in every repo. A
> `Either ANTHROPIC_API_KEY, CLAUDE_CODE_OAUTH_TOKEN, ... is required` message is a
> different problem: the secret is missing or empty in that one repo.

### 2. Add the caller workflows

Create one small caller per audit under `.github/workflows/` in the new repo.
They all follow the same shape — `schedule` + `workflow_dispatch`, the right
`permissions`, and a **SHA-pinned** `uses:` (see [Versioning](#versioning) for how to
get the SHA). Minimal example (`quality.yml`):

```yaml
name: Quality
on:
  schedule:
    - cron: '17 6 * * 2'   # Tuesday 06:17 — pick a per-repo minute, see Scheduling
  workflow_dispatch:
permissions:
  contents: read
  issues: write
  id-token: write        # required: claude-code-action authenticates via OIDC — never remove
jobs:
  audit:
    uses: fewlme/workflows/.github/workflows/code-quality.yml@<40-char-sha>  # v1
    # Pass only the secrets the reusable workflow declares — never `secrets: inherit`.
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

Swap `code-quality.yml` for whichever audit you want. `pull-requests: read` is
no longer needed by any workflow. Per-workflow notes:

- **`security-audit.yml`** → no extra permissions. It no longer needs
  `security-events: write`: scanner output (`semgrep.sarif`, Gitleaks
  `results.sarif`, `composer-audit.json`, `npm-audit.json`, `pip-audit.json`,
  `trivy-report.json`, `tool-status.txt`) is kept as a run artifact
  (`security-audit-reports-<run_id>`, 30 days) and triaged by Claude into
  issues. Runs on `claude-opus-5` by default and has the largest turn budget
  (`max_turns: 320`, 120-minute job timeout).

- **`docs.yml`** → needs `contents: write` **and** `pull-requests: write`
  **unconditionally**. The reusable job declares them, and a called workflow
  cannot elevate beyond its caller, so the minimal snippet above fails
  validation for `docs.yml` even with `create_pr: false`. Full caller:

  ```yaml
  name: Docs
  on:
    schedule:
      - cron: '23 6 * * 4'   # Thursday 06:23
    workflow_dispatch:
  permissions:
    contents: write        # the reusable job declares it — required even with create_pr: false
    issues: write
    pull-requests: write   # same
    id-token: write        # required: claude-code-action authenticates via OIDC — never remove
  jobs:
    audit:
      uses: fewlme/workflows/.github/workflows/docs.yml@<40-char-sha>  # v1
      with:
        create_pr: false   # true = also open a PR with mechanical fixes (typos, stale commands, .env.example keys)
      secrets:
        CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  ```

  A PR opened with the job token does not trigger the consumer's
  `on: pull_request` workflows; CI on the docs PR runs after a human pushes to
  it.

- **`legal-compliance.yml`** → no extra permissions (issue-only, like
  `code-quality.yml`). Legal surfaces churn slowly, so a **quarterly** schedule
  keeps noise low — but the cadence is caller-controlled, set it to whatever fits.
  The 3rd of the month keeps it off `dependency-health.yml`'s 1st:

  ```yaml
  name: Legal
  on:
    schedule:
      - cron: '0 6 3 */3 *'   # quarterly, 3rd of the month 6am
    workflow_dispatch:
  permissions:
    contents: read
    issues: write
    id-token: write        # required: claude-code-action authenticates via OIDC — never remove
  jobs:
    audit:
      uses: fewlme/workflows/.github/workflows/legal-compliance.yml@<40-char-sha>  # v1
      secrets:
        CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  ```

  Note: findings are a screening aid, not legal advice — every issue carries a
  "confirm with counsel" disclaimer. When a `composer.json` is present the
  workflow also runs `composer licenses` and hands the result to Claude.

- **`ui-ux.yml`** → no extra permissions (issue-only, like `code-quality.yml`).
  Audits design consistency, UX heuristics, and whether the project makes the
  most of the UI library it already uses (it fetches the installed version's
  docs to judge this). Skips itself when no frontend framework, UI/CSS library,
  HTML page or server-rendered template layer (Blade/Twig/ERB/Jinja) is
  detected. Suggested cadence: monthly on the
  8th (`cron: '0 6 8 * *'`).

- **`seo.yml`** → no extra permissions (issue-only, like `code-quality.yml`).
  Audits technical SEO (meta, canonicals, sitemap/robots, structured data) and
  answer-engine readiness (llms.txt, AI-crawler robots policy, extractable
  content). Alongside that technical audit it runs an **editorial copy-suggestions
  pass** on a higher-reasoning model (`claude-opus-5` by default) that files ONE
  consolidated `[SEO][editorial]` report issue per calendar month with
  before/after rewrites for titles, meta descriptions, headings and answer text.
  The editorial pass is on by default — disable it with `editorial: false`, change
  its model with `editorial_model`, or its turn cap with `editorial_max_turns`.
  Pass `site_url` to also verify the live site (robots.txt, sitemap, rendered
  pages); without it the audit is code-only. Skips itself on non-web projects.
  Suggested cadence: monthly on the 22nd:

  ```yaml
  jobs:
    audit:
      uses: fewlme/workflows/.github/workflows/seo.yml@<40-char-sha>  # v1
      with:
        site_url: https://example.com
      secrets:
        CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  ```

- **`rd-ideas.yml`** → no extra permissions (issue-only, like `code-quality.yml`).
  It scouts the codebase, the git history and the web for new-feature
  opportunities and files ONE curated `[R&D] Ideas report - <YYYY-MM>` issue per
  calendar month (a second run in the same month skips creation, even if the
  report was closed). A monthly cadence works well, offset from
  `dependency-health.yml` on the 1st:

  ```yaml
  name: R&D Ideas
  on:
    schedule:
      - cron: '0 6 15 * *'   # monthly, 15th 6am
    workflow_dispatch:
  permissions:
    contents: read
    issues: write
    id-token: write        # required: claude-code-action authenticates via OIDC — never remove
  jobs:
    audit:
      uses: fewlme/workflows/.github/workflows/rd-ideas.yml@<40-char-sha>  # v1
      secrets:
        CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  ```

- **`config-drift.yml`** → nothing extra: no extra permissions, no inputs
  beyond the common ones. It inventories every tracked env example file,
  compose file, Dockerfile, `next.config.*` and CodeIgniter `app/Config/*.php`,
  runs dotenv-linter, `docker compose config`, hadolint and ripgrep over them,
  and has Claude diff env keys used in code against the examples and compose
  files, check compose validity and Dockerfile hygiene, and flag Node/PHP
  version skew across manifests, Dockerfiles and CI. Env **values** are never
  printed — key names only. Skips itself when no configuration surface exists.
  Suggested cadence: monthly on the 5th (`cron: '0 6 5 * *'`).

- **`live-site-ops.yml`** → no extra permissions, but it needs **`site_urls`**
  (comma-separated `https://` URLs, one per public host) — without it the job
  skips with a note. Optional **`deep_tls: true`** also runs testssl.sh (adds
  2-4 minutes per host). The probes are read-only GETs against the listed hosts
  plus `dns.google` and `rdap.org`; Claude then separates "never configured"
  from "deploy drift" by reading the code-side header/redirect config and
  checks the mail providers used in code against live SPF. Suggested cadence:
  weekly, e.g. Sunday 05:37:

  ```yaml
  name: Live site ops
  on:
    schedule:
      - cron: '37 5 * * 0'   # Sunday 05:37
    workflow_dispatch:
  permissions:
    contents: read
    issues: write
    id-token: write        # required: claude-code-action authenticates via OIDC — never remove
  jobs:
    audit:
      uses: fewlme/workflows/.github/workflows/live-site-ops.yml@<40-char-sha>  # v1
      with:
        site_urls: https://example.com,https://admin.example.com
        deep_tls: false
      secrets:
        CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  ```

The fastest way to bootstrap is to copy the caller files from an existing
project (e.g. `bilbokidmodels` or `bluelift`) and keep the SHA pins as-is —
Dependabot (next step) bumps them for you.

### 3. Add Dependabot (required)

The pins are only safe if something keeps them fresh. Add
`.github/dependabot.yml` to the new repo so the `github-actions` ecosystem is
watched — Dependabot then opens a PR whenever the pinned SHA falls behind `v1`.
Group the `fewlme/workflows` pins so all callers bump in **one** PR (this is
what `bluelift` uses):

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    commit-message:
      prefix: "ci"
    groups:
      fewlme-workflows:
        patterns:
          # Reusable workflows are named by full path
          # (fewlme/workflows/.github/workflows/docs.yml), so the glob is required
          # — a bare "fewlme/workflows" matches nothing and grouping silently no-ops.
          - "fewlme/workflows*"
```

The grouped PR lands on a `dependabot/github_actions/fewlme-workflows-<hash>`
branch — that name is what the optional auto-merge workflow matches (see
[Versioning](#versioning)).

### 4. Trigger a first run

```bash
gh workflow run quality.yml --repo <owner>/<repo>
```

## Scheduling

Every consumer repo runs on the **same Claude subscription**, and a subscription
has one shared 5-hour usage window with **no overage**: when it is exhausted the
Claude step fails with `overageStatus rejected` / `api_retry` messages and the
audits queued behind it fail too. So do not copy identical crons into every
repo — stagger them:

- Give each repo its own **minute** offset and avoid `:00` (GitHub delays
  top-of-the-hour crons the most anyway): `17 6 * * 1` in one repo,
  `41 6 * * 1` in the next.
- Spread the heavy audits across **hours** as well, e.g. the security audit at
  `17 6 * * 1`, `17 12 * * 1` and `17 18 * * 1` in three repos rather than
  three copies of `0 7 * * 1`.
- Keep the monthly reports (`dependency-health` on the 1st, `config-drift` on
  the 5th, `ui-ux` on the 8th, `rd-ideas` on the 15th, `seo` on the 22nd) on
  different days, and put the quarterly legal audit on the 3rd.
- Cron runs in UTC by default; set `on.schedule[].timezone` (e.g.
  `timezone: Europe/Paris`) on a schedule entry if you want local time.

## Inputs

All audits accept these (passed under `with:` in the caller). Passing an input
a workflow does not declare fails the caller's validation, so check the scope
column:

| Input | Type | Default | Notes |
|---|---|---|---|
| `severity_threshold` | string | `high` | Min severity to file issues for (`critical`/`high`/`medium`/`low`). **Not accepted** by `dependency-health` / `docs` / `rd-ideas` — passing it fails workflow validation. In `code-quality` / `performance` / `legal-compliance` / `security-audit` / `config-drift` / `live-site-ops`, findings below the threshold are still listed in a collapsed "Below threshold" block of the run summary. |
| `create_issues` | boolean | `true` | Set `false` for a dry run (summary only, no issues). |
| `claude_model` | string | `claude-sonnet-5` (`claude-opus-5` for `security-audit`) | Model used for the audit; `claude-sonnet-5` is also the fallback model everywhere. For large monorepos a caller can pass `claude-sonnet-5[1m]` (1M-token context). |
| `effort` | string | `high` | Reasoning effort for the Claude session: `low`/`medium`/`high`/`xhigh`/`max`. |
| `max_turns` | number | `240` (`180` for `dependency-health` / `rd-ideas`, `320` for `security-audit`) | Turn cap for the Claude session. Under claude-code-action >= 1.0.188 the step also fails when the reported `num_turns` (= tool calls + 1) exceeds it, so keep ~3x the intended number of round trips. The run summary's telemetry table shows `num_turns/max_turns`. |
| `create_pr` | boolean | `false` | **`docs.yml` only** — open a PR for mechanical fixes. |
| `site_url` | string | `''` | **`seo.yml` only** — deployed-site URL enabling live robots/sitemap/rendered-page checks. |
| `editorial` | boolean | `true` | **`seo.yml` only** — run the editorial copy-suggestions pass (one consolidated report issue per month). Set `false` to skip it. |
| `editorial_model` | string | `claude-opus-5` | **`seo.yml` only** — model for the editorial pass (the technical audit still uses `claude_model`). |
| `editorial_max_turns` | number | `180` | **`seo.yml` only** — turn cap for the editorial Claude session (same `num_turns` semantics as `max_turns`). |
| `site_urls` | string | `''` | **`live-site-ops.yml` only** — comma-separated `https://` URLs to scan, one per public host (`https://example.com,https://admin.example.com`). Required in practice: the job skips without it. |
| `deep_tls` | boolean | `false` | **`live-site-ops.yml` only** — also run testssl.sh (adds 2-4 min per host). |

## Issue labels

Each audit ensures its labels exist before filing issues (an idempotent
`gh label create --force` step), so a brand-new repo doesn't need any labels
pre-created. `gh issue create` rejects the whole command if any label is
missing, so this step is what keeps findings labelled.

Issues are authored by **`github-actions[bot]`**: the Claude step runs `gh`
on the job token (scoped by the caller's `permissions:`), not on a Claude
GitHub App token. `dependency-health`, `rd-ideas` and the `seo` editorial pass
file **one report per calendar month** (title suffix `- YYYY-MM`); a second run
in the same month skips creation even when that month's report is already
closed.

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
| seo | `seo` + severity, plus `editorial` on the editorial report issue |
| config-drift | `config` + severity |
| live-site-ops | `ops` + severity |

## Versioning

Consumers pin a **full 40-char commit SHA**, not the moving `v1` tag, so a
compromised or accidental push to this repo cannot execute in every consumer on the
next run (CWE-829). The `v1` tag still exists as a human-readable "latest" marker and
as the target Dependabot tracks — each caller writes `...@<sha>  # v1`, and Dependabot
bumps the SHA toward `v1`.

The consumers auto-merge that bump (see below), so the real security gates are
**review on this repo's `master`** and the **deliberate `v1` move**: nothing
reaches a consumer until a maintainer moves the tag.

**Get the SHA to pin** (the commit `v1` currently points to):

```bash
gh api repos/fewlme/workflows/commits/v1 --jq .sha
```

**Maintainers — after merging a change to `master`, move the tag** so Dependabot
picks it up. This includes Dependabot's own merges in this repo (the
`chore(deps): bump ...` PRs) — a merged action bump does nothing for consumers
until `v1` moves:

```bash
git fetch origin && git tag -f v1 origin/master && git push -f origin v1
```

Verify that `v1` matches `master`:

```bash
test "$(gh api repos/fewlme/workflows/git/ref/tags/v1 --jq .object.sha)" = "$(gh api repos/fewlme/workflows/commits/master --jq .sha)"
```

Consumers do **not** update instantly — they update when their Dependabot PR merges.
Existing open issues are not retro-labelled; changes take effect on the next
scheduled or dispatched run after a consumer repins.

### Optional: auto-merge the pin bump

`bluelift`, `cigalio`, `welf` and `bilbokidmodels` ship an `automerge.yml` that
squash-merges the weekly `fewlme/workflows` Dependabot PR without a review. It
is safe there because every workflow in those repos is `schedule` /
`workflow_dispatch` — nothing runs on `pull_request`, so there is no check to
wait for and GitHub's own auto-merge would have nothing to gate on. It matches
**only** the fewlme/workflows bump (the grouped branch name from the Dependabot
config above, plus the per-file form for stragglers); third-party action bumps
keep manual review, since an unreviewed pin move is exactly the supply-chain case
SHA pinning exists for:

```yaml
name: Dependabot auto-merge
on: pull_request_target
permissions:
  contents: write
  pull-requests: write
jobs:
  merge:
    if: >-
      github.actor == 'dependabot[bot]' &&
      (startsWith(github.head_ref, 'dependabot/github_actions/fewlme-workflows-') ||
       startsWith(github.head_ref, 'dependabot/github_actions/fewlme/workflows/'))
    runs-on: ubuntu-latest
    steps:
      - run: gh pr merge "$PR_URL" --squash --delete-branch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_URL: ${{ github.event.pull_request.html_url }}
```

The job never checks out PR code, so `pull_request_target` is only used for its
write-capable token. zizmor still flags the `github.actor` gate as spoofable;
`github.event.pull_request.user.login == 'dependabot[bot]'` is the stricter
form if you want the finding gone.

Be explicit about what this does: it **removes the review gate in the consumer
repo**. Whatever `v1` points to runs in that repo with the caller's permissions
after the next Dependabot cycle, so the review on this repo's `master` and the
manual `v1` move are the only checks left.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Either ANTHROPIC_API_KEY, CLAUDE_CODE_OAUTH_TOKEN, ... is required` | The secret is missing or empty in that repo (this is **not** the expired-token symptom) | `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>` |
| Claude step fails seconds after it starts with a `401` / an `is_error` result — in every repo the same day | The OAuth token has expired (~1 year) | `claude setup-token`, then update the secret in every repo |
| `exceeding the configured maximum of N turns` | Post-hoc guard in claude-code-action >= 1.0.188: `num_turns` (tool calls + 1) went over `max_turns`. Everything the session already wrote (issues, summary) is persisted; only the step status is failed. The telemetry step warns at 80% of the cap | Raise `max_turns` (or `editorial_max_turns`) in the caller |
| `overageStatus rejected` / `api_retry` in the Claude step log | The subscription's shared 5-hour usage window is exhausted — several repos ran at once and there is no overage | Stagger the crons across repos (see [Scheduling](#scheduling)); re-run later with `gh workflow run` |
| Slack says `FAILED` | The Slack step reports the **Claude step outcome**: anything but `success` (turn cap, auth, rate limit) is FAILED, even when the job itself finished | Open the run link; the `Run telemetry` table in the summary shows `num_turns/max_turns` and the result subtype |
| Caller fails validation with a permissions error for `docs.yml` | The reusable job declares `contents: write` + `pull-requests: write` and a called workflow cannot elevate beyond its caller | Grant both in the caller, regardless of `create_pr` |
| Caller fails validation with an input not defined in the referenced workflow | `severity_threshold` passed to `dependency-health` / `docs` / `rd-ideas`, or a workflow-specific input (`site_url`, `site_urls`, `deep_tls`, `editorial*`, `create_pr`) passed to the wrong workflow | Remove the input (see the scope column in [Inputs](#inputs)) |
| Issues created without labels | Labels missing **and** running an old pin without the label-creation step | Merge the Dependabot PR (or manually repin) to the SHA `v1` points to |
| A fix isn't taking effect | Consumer pinned to an old SHA, or `v1` was never moved after the merge | Move `v1` (see [Versioning](#versioning)), then merge the pending Dependabot bump or repin to the SHA from `gh api repos/fewlme/workflows/commits/v1 --jq .sha` |
| Audit ran but filed nothing | No findings at `severity_threshold`, `create_issues: false`, the audit skipped itself (no frontend / web project / config surface / `site_urls`), or a monthly report already exists for this month | Lower the threshold / check the run summary |
| A scanner report is missing or empty in `security-audit` / `dependency-health` | That scan failed; the Claude prompt treats a missing, 0-byte or `error` report as **failed**, never as clean | Check `tool-status.txt` (rc + byte count per scanner) in the `*-reports-<run_id>` artifact and the job log |

## Notes

- **Consumer hooks are disabled during audits.** The Claude step passes
  `settings: '{"disableAllHooks": true}'`, so a consumer repo's
  `.claude/settings.json` hooks and status-line commands do not run inside the
  audit session.
- **Actions logs contain unredacted tool output.** `show_full_output` is on so
  the log shows every tool call and result. The prompts forbid printing secret
  values and env values, but treat the logs as sensitive — set
  `show_full_output: false` in a fork if outside collaborators can read the logs.
- **`persist-credentials: false` on every checkout.** `gh` uses the job token
  via `GH_TOKEN`, and claude-code-action configures its own git auth; nothing is
  left in `.git/config`. The Claude step also runs `gh`/`git` on the job token
  (`github_token: ${{ secrets.GITHUB_TOKEN }}`), scoped by the caller's
  `permissions:`, instead of the Claude GitHub App token the action would
  otherwise mint.
- **Read-only by construction.** Every audit except `docs.yml` denies
  `Edit`/`Write`, `git push`, `gh api`/`gh pr`/`gh secret`/`gh repo`/
  `gh workflow`, `curl`/`wget` and issue close/edit/delete to the Claude
  session; `docs.yml` can push its `docs/auto-fix-<date>` branch and open a PR
  but is denied force-pushes, pushes to `master`/`main` and `gh pr merge`.
- **Single session, deterministic.** `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS=1`
  and `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1` back the "do everything in this
  session" rule in the prompts; findings are filed as soon as they are confirmed,
  highest severity first, so a run cut short by the turn cap still leaves the
  most serious issues behind.
- **Scanner reports are artifacts.** `security-audit` and `dependency-health`
  upload their raw reports plus `tool-status.txt` as
  `<workflow>-reports-<run_id>` (30 days); that artifact is the place to look
  when the summary lists a scanner as failed.
- **Pin refresh routine.** The SHA-pinned `uses:` lines in this repo are owned
  by Dependabot (`.github/dependabot.yml`: weekly, 7-day cooldown, minor/patch
  bumps grouped into one PR, majors separate). The periodic refresh of prompts
  and steps in these workflows never edits a `uses:` line by hand; after either
  kind of merge, move `v1` (see [Versioning](#versioning)) so consumers pick it up.
