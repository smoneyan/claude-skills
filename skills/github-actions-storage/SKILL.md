---
name: github-actions-storage
description: Analyze GitHub Actions artifact storage usage across an entire organization or from a billing CSV report — identify top consuming repositories, calculate costs, and build a cleanup plan. Trigger when user asks about org-wide Actions storage costs, wants to analyze a billing report CSV, asks which repositories are using the most storage, wants to reduce GitHub Actions bill, or asks about storage across multiple repositories.
---

# GitHub Actions Storage Analysis

When this skill is triggered, perform a live analysis using `gh` CLI commands or process a billing CSV the user provides. Do not just provide documentation — execute the analysis and return real findings.

## Determine Mode

Ask or infer from context:
- **Mode A — Billing CSV**: User has a GitHub billing report CSV → parse it for storage data
- **Mode B — Live Org Query**: User provides an org name → query via `gh` CLI

> **Preferred for GitHub Enterprise**: Use the `github-billing-usage-reports` skill to generate and download a billing CSV, then analyze it here (Mode A). The enterprise slug and org name differ — the live org query (Mode B) won't work against an enterprise slug directly.

---

## Mode A: Billing CSV Analysis

If the user provides a CSV file path, analyze it directly:

```bash
# Quick check: what's the total storage and top repos?
# The CSV has columns: Date, Product, Repository Slug, Quantity, Unit Type, Price Per Unit, Multiplier, Owner
# Filter for 'Actions' product and storage unit types

# Show top consuming repos
 awk -F',' 'NR==1{print; next} $2~/Actions/ && $6~/storage/{print}' billing.csv \
  | sort -t',' -k4 -rn | head -20
```

Or instruct the user to run this Python one-liner (requires no dependencies beyond stdlib):
```bash
python3 -c "
import csv, sys
from collections import defaultdict
data = defaultdict(float)
with open('$CSV_PATH') as f:
    for row in csv.DictReader(f):
        if 'storage' in row.get('Unit Type','').lower() or 'storage' in row.get('SKU','').lower():
            repo = row.get('Repository Slug') or row.get('repository','unknown')
            qty = float(row.get('Quantity') or row.get('quantity') or 0)
            data[repo] += qty
for repo, qty in sorted(data.items(), key=lambda x: -x[1])[:20]:
    print(f'{qty/24:.1f} GB avg\t{repo}')
"
```

**Key Metrics to Extract:**
- GB-hours per repository (raw quantity) — the `unit_type` for `actions_storage` SKU is `gigabyte-hours`, NOT GB-day
- Average daily GB = GBh ÷ 24
- Percentage of total org storage
- Estimated monthly cost at $0.008/GB-hour (often fully discounted to $0 on Enterprise plans)

**Present as:**
```
## Org Storage Analysis from Billing Report

Total storage: X,XXX GB-hours (avg X.X GB/day)
Estimated monthly cost: $XXX

### Top Repositories by Storage

| Rank | Repository | Avg GB | GB-Hours | % Total | Est. Monthly |
|------|-----------|--------|----------|---------|--------------|
| 1    | org/repo  | XX.X   | X,XXX    | XX%     | $XXX         |
```

---

## Mode B: Live Org Query via gh CLI

If the user provides an org name, query the top repos:

```bash
# List repos with most artifact storage (samples top repos)
ORG="{org}"

# Get list of repos
gh repo list $ORG --limit 100 --json name,updatedAt \
  --jq 'sort_by(-.updatedAt) | .[].name' | head -30 | while read repo; do
    count=$(gh api "repos/$ORG/$repo/actions/artifacts?per_page=1" --jq '.total_count' 2>/dev/null || echo 0)
    echo "$count $repo"
done | sort -rn | head -15
```

Then for the top repos, get detailed storage:
```bash
# Get total artifact size for a specific repo
gh api "repos/{org}/{repo}/actions/artifacts?per_page=100" --paginate \
  --jq '[.artifacts[].size_in_bytes] | add // 0 | . / 1073741824 | . * 100 | round / 100 | tostring + " GB"'
```

**Note**: Live querying is rate-limited. Process repos in batches and cache results.

---

## Phase 2.5: Check Expired vs Unexpired Artifacts

**Critical step before recommending cleanup** — billing storage numbers are dominated by expired artifacts pending GitHub's async garbage collection. Always check the live artifact status for the top repos:

```bash
gh api "repos/{org}/{repo}/actions/artifacts?per_page=100" --paginate \
  --jq '[.artifacts[] | {name, size_in_bytes, created_at, expires_at, expired}]' \
  > /tmp/artifacts.json
```

Then analyze with Python:
```python
import json, re
from datetime import datetime, timezone

with open('/tmp/artifacts.json') as f:
    content = f.read().strip()

all_artifacts = []
for block in re.split(r'\]\s*\[', content.strip('[]')):
    try:
        all_artifacts.extend(json.loads('[' + block + ']'))
    except:
        pass

now = datetime.now(timezone.utc)
expired = [a for a in all_artifacts if a.get('expired')]
unexpired = [a for a in all_artifacts if not a.get('expired')]

print(f"Expired:   {len(expired)} artifacts  |  {sum(a['size_in_bytes'] for a in expired)/1024/1024:.0f} MB")
print(f"Unexpired: {len(unexpired)} artifacts  |  {sum(a['size_in_bytes'] for a in unexpired)/1024/1024:.0f} MB")

for a in sorted(unexpired, key=lambda x: -x['size_in_bytes'])[:15]:
    exp = a.get('expires_at') or 'N/A'
    days_left = ''
    if exp != 'N/A':
        exp_dt = datetime.fromisoformat(exp.replace('Z', '+00:00'))
        days_left = f"  ({int((exp_dt - now).total_seconds()/86400)}d left)"
    print(f"  {a['size_in_bytes']/1024/1024:>8.1f} MB  created={a['created_at'][:10]}  expires={exp[:10]}{days_left}  {a['name']}")
```

**Interpreting results:**
- If most storage is **expired** → GitHub GC will reclaim it automatically; no immediate action needed
- If most storage is **unexpired** → investigate artifact type and retention-days setting
- **Common culprit**: Playwright E2E test reports are 100–160 MB each and accumulate fast; 10-day retention × many parallel runs = GBs of live storage

---

## Phase 3: Action Plan

After identifying the top consumers, generate a prioritized action plan:

### Priority Matrix

| Storage | Age >90d | Priority | Action |
|---------|----------|----------|--------|
| >50 GB  | Any      | 🔥 Critical | Immediate cleanup |
| >10 GB  | >50%     | 🔴 High | Cleanup this week |
| >10 GB  | <50%     | 🟡 Medium | Add retention policy |
| <10 GB  | >75%     | 🟡 Medium | Review workflow efficiency |
| <10 GB  | <75%     | 🟢 Low | Monitor |

### For Each High-Priority Repo

Offer to run the `github-repo-artifact-analysis` skill for a deep dive, or generate bulk cleanup commands:

```bash
# Estimate cleanup potential for a repo
gh api "repos/{org}/{repo}/actions/artifacts?per_page=100" --paginate \
  --jq '[.artifacts[] | select(.created_at < "'$(date -v-90d +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -d "90 days ago" +%Y-%m-%dT%H:%M:%SZ)'") | .size_in_bytes] | add // 0 | . / 1073741824 | . * 10 | round / 10 | tostring + " GB can be freed"'
```

### Organization-Level Policy Recommendations

1. **Default retention** — enforce via branch protection or workflow templates:
   ```yaml
   - uses: actions/upload-artifact@v4
     with:
       retention-days: 30  # Set org default
   ```
   **For Playwright/E2E test reports specifically** — these are large (100–160 MB each) and rarely needed after a few days. Use 3–5 days max:
   ```yaml
   - uses: actions/upload-artifact@v4
     with:
       name: playwright-report
       path: playwright-report/
       retention-days: 3
   ```

2. **Automated cleanup workflow** — add to `.github/workflows/artifact-cleanup.yml`:
   ```yaml
   name: Artifact Cleanup
   on:
     schedule:
       - cron: '0 2 * * 0'  # Weekly Sunday 2AM
     workflow_dispatch:
   jobs:
     cleanup:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/github-script@v7
           with:
             script: |
               const cutoff = new Date(Date.now() - 30 * 86400000);
               const { data: { artifacts } } = await github.rest.actions.listArtifactsForRepo({
                 owner: context.repo.owner, repo: context.repo.repo, per_page: 100
               });
               for (const a of artifacts) {
                 if (new Date(a.created_at) < cutoff) {
                   await github.rest.actions.deleteArtifact({
                     owner: context.repo.owner, repo: context.repo.repo, artifact_id: a.id
                   });
                   console.log(`Deleted: ${a.name}`);
                 }
               }
   ```

3. **Cost calculation**:
   ```
   Monthly cost = (avg_daily_GB × 30 × $0.008) for public repos
   Free tier: 500 MB (Free), 2 GB (Pro), 50 GB (Team), unlimited (Enterprise)
   ```

## Troubleshooting

- **Rate limited**: Add `--header 'Accept: application/vnd.github+json'` and use a PAT with higher limits
- **No artifacts found**: Check token has `repo` + `actions:read` scopes
- **CSV format varies**: GitHub changes billing CSV columns over time — inspect headers first with `head -1 billing.csv`
