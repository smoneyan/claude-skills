# github-actions-storage

Analyze GitHub Actions artifact storage usage across an entire organization or from a billing CSV report — identify top consuming repositories, calculate costs, and build a cleanup plan.

## Usage

This skill is automatically triggered when you ask Claude about:
- Org-wide Actions storage costs
- Analyzing a GitHub billing report CSV
- Which repositories are using the most storage
- Reducing your GitHub Actions bill
- Storage costs across multiple repositories

## Modes

**Mode A — Billing CSV**: Provide a path to your GitHub billing CSV export and the skill will parse it for storage data, rank repositories by usage, and estimate costs.

**Mode B — Live Org Query**: Provide an org name and the skill queries the GitHub API via `gh` CLI to identify top storage consumers in real time.

## Requirements

- `gh` CLI installed and authenticated (`gh auth login`)
- Token scopes: `repo` + `actions:read`
