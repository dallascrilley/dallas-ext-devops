---
name: github-ci-setup
description: Use when setting up a new repository with standard GitHub CI infrastructure, adding PR templates, configuring validation workflows, creating required labels, or adding Claude code review automation
---

# GitHub CI Setup

## Overview

Scaffolds a repository with standard GitHub CI/CD infrastructure: PR template with merge-order metadata, label and body validation workflow, and Claude-powered code review. All components are idempotent and safe to re-run.

## Quick Reference

| Component | Target Path | Purpose |
|-----------|-------------|---------|
| PR Template | `.github/PULL_REQUEST_TEMPLATE.md` | Merge-order metadata |
| PR Conventions | `.github/workflows/pr-conventions.yml` | Label + body validation |
| Auto Size Label | `.github/workflows/pr-size-label.yml` | Auto-apply size:XS-XL from diff |
| PR Title Check | `.github/workflows/pr-title.yml` | Enforce conventional commits |
| Stale PRs | `.github/workflows/stale.yml` | Auto-label/close stale PRs |
| Changelog | `.github/workflows/changelog.yml` + `cliff.toml` | Auto-generate CHANGELOG.md |
| Claude Review | `.github/workflows/pr-claude-code-review.yml` | AI code review |
| Labels | Repository settings | Priority/risk/size |
| Branch Protection | Repository settings | Require checks + reviews |

## Prerequisites

- `gh` CLI authenticated with repo admin access
- `ANTHROPIC_API_KEY` repository secret (for Claude review)

## Setup Workflow

### Step 1: Scaffold directories

```bash
mkdir -p .github/workflows
```

### Step 2: Check for existing files

Before writing each file, check if it already exists. If it does, ask the user whether to overwrite or merge.

```bash
for f in .github/PULL_REQUEST_TEMPLATE.md .github/workflows/pr-conventions.yml .github/workflows/pr-size-label.yml .github/workflows/pr-title.yml .github/workflows/stale.yml .github/workflows/changelog.yml cliff.toml .github/workflows/pr-claude-code-review.yml; do
  [ -f "$f" ] && echo "EXISTS: $f"
done
```

### Step 3: Create PR template

Write `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Merge-order metadata

<!-- Add matching labels (P0-P3, risk:*, size:*, blocked/ready-to-merge) so the queue can sort. -->

- **Priority:** [ ] P0  [ ] P1  [ ] P2  [ ] P3
- **Risk:** [ ] low  [ ] med  [ ] high
- **Dependencies:** Depends on #____  /  Blocked by #____  _(leave blank if none)_
- **Merge notes:** _(only when needed)_ Must merge before/after ____
- **Rollback plan:** _(required for risk:high)_ ____
- **Linked issue (GitHub):** #____  _(optional)_
- **Linked issue (Linear):** ____  _(optional)_
- **Linked issue (td):** ____  _(optional)_

---

## Summary

_(What this PR does and why.)_

## Verification

_(How to confirm it works; e.g. `just check`, manual steps.)_
```

### Step 4: Create PR conventions workflow

Write `.github/workflows/pr-conventions.yml`:

```yaml
name: PR Conventions

on:
  pull_request:
    types: [opened, edited, labeled, unlabeled, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  issues: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const pr = context.payload.pull_request;
            const labels = (pr.labels || []).map(l => l.name);

            function hasPrefix(prefix) {
              return labels.some(l => l.startsWith(prefix));
            }

            const hasP = hasPrefix('P');
            const hasRisk = hasPrefix('risk:');
            const hasSize = hasPrefix('size:');
            const body = pr.body || '';
            const hasMergeOrder = body.includes('## Merge-order metadata');
            const hasSummary = body.includes('## Summary');
            const hasVerification = body.includes('## Verification');

            const errors = [];
            if (!hasP) errors.push('Missing priority label (P0-P3)');
            if (!hasRisk) errors.push('Missing risk label (risk:low/med/high)');
            if (!hasSize) errors.push('Missing size label (size:XS/S/M/L/XL)');
            if (!hasMergeOrder) errors.push('Missing "## Merge-order metadata" section');
            if (!hasSummary) errors.push('Missing "## Summary" section');
            if (!hasVerification) errors.push('Missing "## Verification" section');

            if (errors.length > 0) {
              core.setFailed('PR validation failed:\n' + errors.join('\n'));
            }
```

### Step 5: Create auto size label workflow

Write `.github/workflows/pr-size-label.yml`:

Automatically applies `size:XS` through `size:XL` based on total lines changed. Removes stale size labels when the diff changes.

```yaml
name: PR Size Label

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const { additions, deletions } = context.payload.pull_request;
            const total = additions + deletions;

            const sizes = [
              { label: 'size:XS', max: 10 },
              { label: 'size:S',  max: 50 },
              { label: 'size:M',  max: 200 },
              { label: 'size:L',  max: 500 },
              { label: 'size:XL', max: Infinity },
            ];

            const target = sizes.find(s => total <= s.max).label;
            const current = context.payload.pull_request.labels
              .map(l => l.name)
              .filter(n => n.startsWith('size:'));

            // Remove stale size labels
            for (const old of current) {
              if (old !== target) {
                await github.rest.issues.removeLabel({
                  ...context.repo,
                  issue_number: context.issue.number,
                  name: old,
                });
              }
            }

            // Add correct label if not already present
            if (!current.includes(target)) {
              await github.rest.issues.addLabels({
                ...context.repo,
                issue_number: context.issue.number,
                labels: [target],
              });
            }

            core.info(`${total} lines changed → ${target}`);
```

### Step 6: Create PR title validation workflow

Write `.github/workflows/pr-title.yml`:

Enforces conventional commit format on PR titles. Important for squash-merge repos where the PR title becomes the commit message.

```yaml
name: PR Title

on:
  pull_request:
    types: [opened, edited, reopened]

permissions:
  pull-requests: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const title = context.payload.pull_request.title;
            // type(optional-scope): description
            const pattern = /^(feat|fix|chore|docs|style|refactor|perf|test|build|ci|revert)(\(.+\))?!?: .+/;

            if (!pattern.test(title)) {
              core.setFailed(
                `PR title "${title}" does not match conventional commits format.\n` +
                `Expected: type(scope): description\n` +
                `Types: feat, fix, chore, docs, style, refactor, perf, test, build, ci, revert`
              );
            }
```

### Step 7: Create stale PR workflow

Write `.github/workflows/stale.yml`:

Labels PRs as `stale` after 14 days of inactivity. Closes them after 7 more days. Exempts PRs with `P0`, `P1`, or `blocked` labels.

```yaml
name: Stale PRs

on:
  schedule:
    - cron: '0 9 * * 1-5'  # Weekdays at 9am UTC

permissions:
  issues: write
  pull-requests: write

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v9
        with:
          days-before-stale: 14
          days-before-close: 7
          stale-pr-label: stale
          stale-pr-message: >
            This PR has been inactive for 14 days. It will be closed in 7 days
            if there is no further activity. Remove the `stale` label or push
            a commit to reset the timer.
          close-pr-message: >
            Closed due to inactivity. Reopen if still needed.
          exempt-pr-labels: 'P0,P1,blocked'
          only-pr: true
          days-before-issue-stale: -1
          days-before-issue-close: -1
```

### Step 8: Create changelog generation workflow

Write `cliff.toml` (git-cliff config):

```toml
[changelog]
header = "# Changelog\n\nAll notable changes to this project will be documented in this file.\n"
body = """
{% if version %}\
    ## [{{ version | trim_start_matches(pat="v") }}] - {{ timestamp | date(format="%Y-%m-%d") }}
{% else %}\
    ## [unreleased]
{% endif %}\
{% for group, commits in commits | group_by(attribute="group") %}
    ### {{ group | striptags | trim | upper_first }}
    {% for commit in commits %}
        - {% if commit.scope %}**{{ commit.scope }}:** {% endif %}\
            {{ commit.message | upper_first }}\
            {% if commit.body %}\n\n{{ commit.body | indent(prefix="  ") }}{% endif %}\
    {%- endfor %}
{% endfor %}\n
"""
trim = true

[git]
conventional_commits = true
filter_unconventional = true
split_commits = false
commit_parsers = [
    { message = "^feat", group = "Features" },
    { message = "^fix", group = "Bug Fixes" },
    { message = "^doc", group = "Documentation" },
    { message = "^perf", group = "Performance" },
    { message = "^refactor", group = "Refactor" },
    { message = "^style", group = "Styling" },
    { message = "^test", group = "Testing" },
    { message = "^chore\\(release\\)", skip = true },
    { message = "^chore", group = "Miscellaneous" },
    { message = "^ci", group = "CI" },
    { message = "^build", group = "Build" },
    { message = "^revert", group = "Reverted" },
]
filter_commits = false
tag_pattern = "v[0-9].*"
sort_commits = "oldest"
```

Write `.github/workflows/changelog.yml`:

```yaml
name: Changelog

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  changelog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate changelog
        uses: orhun/git-cliff-action@v4
        with:
          config: cliff.toml
          args: --verbose
        env:
          OUTPUT: CHANGELOG.md
          GITHUB_REPO: ${{ github.repository }}

      - name: Commit changelog
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          if git diff --quiet CHANGELOG.md 2>/dev/null; then
            echo "No changelog changes"
          else
            git add CHANGELOG.md
            git commit -m "chore: update changelog [skip ci]"
            git push
          fi
```

Notes:
- `[skip ci]` in the commit message prevents an infinite loop of changelog commits triggering more changelog runs
- `fetch-depth: 0` is required so git-cliff can see the full commit history
- The workflow only commits if CHANGELOG.md actually changed

### Step 9: Create Claude code review workflow

Write `.github/workflows/pr-claude-code-review.yml`:

Note: Fork PRs skip review because `ANTHROPIC_API_KEY` is not available to forks. This is intentional — using `pull_request` (not `pull_request_target`) prevents secret exposure.

```yaml
name: PR - Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
  issue_comment:
    types: [created]

concurrency:
  group: ${{ github.workflow }}-pr-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write
  issues: read

jobs:
  review:
    # Run on PR events (same-repo only), or on issue comments that mention @claude
    if: |
      (github.event_name == 'pull_request' &&
       github.event.pull_request.head.repo.full_name == github.repository) ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@claude'))
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository (PR event)
        if: github.event_name == 'pull_request'
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0

      - name: Checkout repository (issue_comment event)
        if: github.event_name == 'issue_comment'
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Checkout PR branch (issue_comment event)
        if: github.event_name == 'issue_comment'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh pr checkout ${{ github.event.issue.number }}

      - name: Get PR base branch
        id: pr-info
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            echo "base_ref=${{ github.event.pull_request.base.ref }}" >> $GITHUB_OUTPUT
          else
            BASE_REF=$(gh pr view ${{ github.event.issue.number }} --json baseRefName -q '.baseRefName')
            echo "base_ref=$BASE_REF" >> $GITHUB_OUTPUT
          fi

      - name: Claude Code Review
        uses: anthropics/claude-code-action@beta
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          model: claude-sonnet-4-20250514
          timeout_minutes: 30
          prompt: |
            Review this Pull Request for code quality, correctness, and security.

            1. Run `git diff origin/${{ steps.pr-info.outputs.base_ref }}...HEAD` to see changes
            2. Review modified files for bugs, security issues, and style problems
            3. Provide feedback organized by severity (Critical, Warning, Suggestion)
          claude_args: |
            --max-turns 10
            --allowedTools "Read,Glob,Grep,Bash(git:*),Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*)"
```

### Step 10: Create required labels

Write `scripts/create-merge-order-labels.sh` and run it. The script is idempotent — it skips labels that already exist.

```bash
#!/usr/bin/env bash
# Create merge-order label taxonomy via GitHub API. Idempotent.
# Usage: ./scripts/create-merge-order-labels.sh [REPO_SLUG]
set -euo pipefail
REPO="${1:-$(gh repo view --json nameWithOwner -q .nameWithOwner)}"
API="repos/${REPO}/labels"
create_or_skip() { local n="$1" c="$2" d="$3"; gh api "GET ${API}/${n}" --silent 2>/dev/null && echo "  skip $n" || { gh api -X POST "$API" -f name="$n" -f color="$c" -f description="$d" --silent; echo "  created $n"; }; }
echo "Merge-order labels for $REPO"
echo "Priority"; create_or_skip "P0" "B60205" "Priority: highest"; create_or_skip "P1" "D93F0B" "Priority: high"; create_or_skip "P2" "FBCA04" "Priority: medium"; create_or_skip "P3" "6E7781" "Priority: low"
echo "Risk"; create_or_skip "risk:low" "0E8A16" "Risk: low"; create_or_skip "risk:med" "FBCA04" "Risk: medium"; create_or_skip "risk:high" "B60205" "Risk: high"
echo "Size"; create_or_skip "size:XS" "E6F6FF" "Size: XS"; create_or_skip "size:S" "C5DEF5" "Size: S"; create_or_skip "size:M" "0052CC" "Size: M"; create_or_skip "size:L" "0747A6" "Size: L"; create_or_skip "size:XL" "172B4D" "Size: XL"
echo "Dependency/state"; create_or_skip "blocked" "B60205" "Blocked; not ready to merge"; create_or_skip "ready-to-merge" "0E8A16" "Ready for merge queue"; create_or_skip "depends-on" "0052CC" "Depends on other PRs (list in body)"; create_or_skip "stacked" "5319E7" "Part of a PR stack"
echo "Release lane"; create_or_skip "release:blocker" "B60205" "Release blocker"; create_or_skip "release:next" "0E8A16" "Target: next release"; create_or_skip "release:later" "6E7781" "Target: later release"
echo "Done."
```

Then run:

```bash
chmod +x scripts/create-merge-order-labels.sh
./scripts/create-merge-order-labels.sh
```

### Step 11: Configure branch protection

Apply branch protection to `main` via the GitHub API. Requires admin access. This makes the validation workflows enforced rather than advisory.

```bash
REPO="$(gh repo view --json nameWithOwner -q .nameWithOwner)"
BRANCH="main"

gh api -X PUT "repos/${REPO}/branches/${BRANCH}/protection" \
  --input - <<'EOF'
{
  "required_status_checks": {
    "strict": true,
    "contexts": ["validate", "label"]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true
  },
  "restrictions": null
}
EOF

echo "Branch protection set on ${BRANCH}"
```

Notes:
- `"strict": true` means the branch must be up-to-date with `main` before merging
- `"enforce_admins": false` lets admins bypass in emergencies — set to `true` for stricter repos
- `"contexts"` lists the required status check names — `validate` is from pr-conventions, `label` is from pr-size-label
- Add `"review"` to contexts if you want Claude review to be required (not recommended — it may timeout)
- For repos that don't need PR reviews (solo projects), remove the `required_pull_request_reviews` block

### Step 12: Add ANTHROPIC_API_KEY secret

The Claude review workflow requires an `ANTHROPIC_API_KEY` repository secret.

```bash
gh secret set ANTHROPIC_API_KEY
```

This will prompt for the key value interactively. If setting non-interactively:

```bash
echo "$KEY" | gh secret set ANTHROPIC_API_KEY
```

### Step 13: Verify

Open a test PR to confirm:
1. The PR template auto-populates with merge-order metadata sections
2. The pr-conventions workflow runs and validates labels + body sections
3. The pr-size-label workflow auto-applies a `size:*` label
4. The pr-title workflow validates conventional commit format
5. The Claude code review workflow triggers on PR open (same-repo PRs only)
6. Branch protection blocks merge until required checks pass
7. After merge, CHANGELOG.md is auto-updated on main

## Customization

| Setting | Default | Notes |
|---------|---------|-------|
| Review model | `claude-sonnet-4-20250514` | Change in `pr-claude-code-review.yml` |
| Review timeout | 30 minutes | Adjust `timeout_minutes` |
| Max review turns | 10 | Adjust `--max-turns` in `claude_args` |
| Review prompt | Generic quality review | Add project-specific checklist or point to an agent file |
| Label colors | Standard scheme | Edit colors in `create-merge-order-labels.sh` |
| Fork PR reviews | Skipped (safe default) | Remove fork guard if you trust all contributors |
| Size thresholds | XS<10, S<50, M<200, L<500, XL>500 | Adjust in `pr-size-label.yml` |
| PR title types | Standard conventional commits | Edit regex in `pr-title.yml` |
| Stale days | 14 days stale, 7 days to close | Adjust in `stale.yml` |
| Stale exemptions | P0, P1, blocked | Add labels to `exempt-pr-labels` |
| Required reviews | 1 approving review | Change `required_approving_review_count` in branch protection |
| Required checks | validate, label | Add/remove from `contexts` array in branch protection |
| Enforce admins | false (admins can bypass) | Set to `true` for stricter enforcement |
| Changelog format | Grouped by conventional commit type | Edit `cliff.toml` body template |
| Changelog trigger | Every push to main | Change branch filter in `changelog.yml` |
