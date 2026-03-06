---
name: github-billing-usage-reports
description: Create a GitHub Enterprise billing usage report export, wait for it to complete, download it, and analyze the results. Use this skill whenever the user wants to analyze GitHub billing data, usage costs, spending by product (Actions minutes, Actions artifact storage, Packages, Copilot, GHAS, Git LFS, etc.), usage by repository, org, or user, or wants to export and inspect a billing usage report. Trigger phrases include "billing report", "usage report", "GitHub costs", "how much are we spending on", "Actions costs", "top consumers", "Actions minutes", "artifact storage", "billing export", "usage export", or any request to understand GitHub spending or consumption across repos or orgs.
---

# GitHub Billing Usage Reports

This skill creates a GitHub Enterprise billing usage report export, polls until ready, downloads the CSV, and analyzes it to answer the user's question.

## Step 1: Gather Context

Determine from the user's request (ask only if truly unclear):

- **Enterprise slug** — e.g., `singapore-press-holdings`
- **Date range** — defaults to the current calendar month if unspecified (`start_date=YYYY-MM-01`, `end_date=today`)
- **What to analyze** — e.g., top repos by Actions minutes, artifact storage, Copilot usage, total spend per product

---

## Step 2: Create the Export

```bash
gh api --method POST \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  /enterprises/{enterprise}/settings/billing/reports \
  -f "report_type=detailed" \
  -f "start_date={YYYY-MM-DD}" \
  -f "end_date={YYYY-MM-DD}"
```

**Response:**
```json
{
  "id": "605788ce-c549-4632-9b8d-70a5d490fd55",
  "report_type": "detailed",
  "start_date": "2026-03-01",
  "end_date": "2026-03-06",
  "status": "processing",
  "created_at": "2026-03-06T06:11:00Z",
  "actor": "smoneyan"
}
```

Capture the `id` for polling.

**report_type options:**
- `detailed` — row per usage event (repo, workflow, user); use for deep analysis
- `summarized` — aggregated totals; use for high-level cost overview

**Token requirement:** The token must have `manage_billing:enterprise` scope. If you get 404, check scopes with `gh auth status` and re-auth: `gh auth refresh -s manage_billing:enterprise`.

---

## Step 3: Poll Until Ready

Poll every 5 seconds. Reports typically complete in 10–60 seconds.

```bash
export_id="{id from step 2}"
enterprise="{enterprise}"

while true; do
  resp=$(gh api \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    /enterprises/$enterprise/settings/billing/reports/$export_id)
  status=$(echo "$resp" | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  echo "Status: $status"
  if [ "$status" = "completed" ] || [ "$status" = "failed" ]; then
    echo "$resp"
    break
  fi
  sleep 5
done
```

If `status` is `"failed"`, inform the user and retry with a narrower date range.

**List existing reports** (to reuse a recent one and avoid creating a duplicate):
```bash
gh api \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  /enterprises/{enterprise}/settings/billing/reports \
  --jq '.usage_report_exports[:5]'
```

---

## Step 4: Download the Report

When `status == "completed"`, the response includes `download_urls` (an array of pre-signed Azure Blob URLs, valid for ~1 hour).

```bash
download_url=$(gh api \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  /enterprises/{enterprise}/settings/billing/reports/{export_id} \
  --jq '.download_urls[0]')

curl -sL "$download_url" -o /tmp/gh_billing_report.csv
head -2 /tmp/gh_billing_report.csv
```

**CSV columns (verified):**
| Column | Description |
|--------|-------------|
| `date` | Billing date |
| `product` | `actions`, `copilot`, `ghas`, `git_lfs`, `packages` |
| `sku` | Specific SKU (see table below) |
| `quantity` | Units consumed |
| `unit_type` | `minutes`, `GB`, `seats`, etc. |
| `applied_cost_per_quantity` | Price per unit (USD) |
| `gross_amount` | `quantity × applied_cost_per_quantity` |
| `discount_amount` | Enterprise discount applied |
| `net_amount` | Actual charge (gross − discount) — **use this for cost analysis** |
| `username` | GitHub username |
| `organization` | GitHub org name |
| `repository` | Repository name (without org prefix) |
| `workflow_path` | Workflow file path (Actions only) |
| `cost_center_name` | Cost center if configured |

**Key SKUs:**
- `actions_linux` — Linux runner minutes
- `actions_macos` — macOS runner minutes (10× multiplier)
- `actions_windows` — Windows runner minutes (2× multiplier)
- `actions_self_hosted_linux` — Self-hosted runner (usually $0)
- `actions_storage` — Actions artifact/cache storage (GB-day)
- `packages_storage` — GitHub Packages storage
- `packages_bandwidth` — Packages data transfer
- `copilot_for_business` — Copilot Business seats
- `copilot_premium_request` — Copilot premium model requests
- `coding_agent_premium_request` — Copilot coding agent requests
- `ghas_licenses` — GitHub Advanced Security seats
- `git_lfs_storage` / `git_lfs_bandwidth` — Git LFS

---

## Step 5: Analyze

Use Python to analyze the CSV. Always use `net_amount` for cost figures (it reflects actual billing after discounts).

```python
import csv
from collections import defaultdict

with open('/tmp/gh_billing_report.csv', newline='', encoding='utf-8-sig') as f:
    reader = csv.DictReader(f)
    rows = list(reader)

print(f"Total rows: {len(rows)}")
print(f"Columns: {reader.fieldnames}")
```

### Top repos by Actions minutes consumed

```python
mins = defaultdict(float)
for r in rows:
    if r['product'] == 'actions' and r['sku'] in ('actions_linux', 'actions_macos', 'actions_windows'):
        mins[f"{r['organization']}/{r['repository']}"] += float(r['quantity'] or 0)

total = sum(mins.values())
print(f"\nTotal Actions minutes: {total:,.0f}")
print("\nTop repos by minutes:")
for repo, qty in sorted(mins.items(), key=lambda x: -x[1])[:20]:
    print(f"  {qty:>8,.0f} mins  ({qty/total*100:.1f}%)  {repo}")
```

### Top repos by Actions artifact storage (GB-day)

```python
storage = defaultdict(float)
for r in rows:
    if r['sku'] == 'actions_storage':
        storage[f"{r['organization']}/{r['repository']}"] += float(r['quantity'] or 0)

total_storage = sum(storage.values())
print(f"\nTotal Actions storage: {total_storage:,.1f} GB-day")
print("\nTop repos by artifact storage:")
for repo, qty in sorted(storage.items(), key=lambda x: -x[1])[:20]:
    cost = sum(float(r['net_amount'] or 0) for r in rows
               if r['sku'] == 'actions_storage'
               and f"{r['organization']}/{r['repository']}" == repo)
    print(f"  {qty:>10,.1f} GB-day  ${cost:>8.2f}  {repo}")
```

### Total cost by product

```python
product_cost = defaultdict(float)
for r in rows:
    product_cost[r['product']] += float(r['net_amount'] or 0)

total_cost = sum(product_cost.values())
print(f"\nTotal enterprise spend: ${total_cost:,.2f}")
for product, cost in sorted(product_cost.items(), key=lambda x: -x[1]):
    print(f"  {product:<20} ${cost:>10,.2f}  ({cost/total_cost*100:.1f}%)")
```

### Actions minutes cost by runner type

```python
runner_cost = defaultdict(lambda: {'minutes': 0, 'cost': 0})
runner_skus = {'actions_linux', 'actions_macos', 'actions_windows', 'actions_self_hosted_linux'}
for r in rows:
    if r['sku'] in runner_skus:
        runner_cost[r['sku']]['minutes'] += float(r['quantity'] or 0)
        runner_cost[r['sku']]['cost'] += float(r['net_amount'] or 0)

for sku, d in sorted(runner_cost.items(), key=lambda x: -x[1]['cost']):
    print(f"  {sku:<35} {d['minutes']:>8,.0f} mins  ${d['cost']:>8.2f}")
```

### Copilot usage by user

```python
copilot_users = defaultdict(float)
for r in rows:
    if r['product'] == 'copilot' and r['sku'] == 'copilot_for_business':
        copilot_users[r['username']] += float(r['quantity'] or 0)
print(f"\nCopilot users: {len(copilot_users)}")
```

---

## Step 6: Present the Report

Structure the output as:

```
## GitHub Billing Report — {enterprise} — {start_date} to {end_date}

**Total enterprise spend**: $X,XXX.XX
**Report type**: Detailed
**Date range**: {start} to {end}

### Actions Minutes — Top Consumers

| Rank | Repository | Minutes | % Total |
|------|-----------|---------|---------|
| 1    | org/repo  | X,XXX   | XX%     |

### Actions Artifact Storage — Top Consumers

| Rank | Repository | GB-Day | Cost |
|------|-----------|--------|------|
| 1    | org/repo  | X,XXX  | $XX  |

### Observations
- ...
```

Always call out standout cases: a repo using >30% of total minutes, zero-cost self-hosted usage, unusual macOS spend, etc.

---

## Troubleshooting

**404 on the reports endpoint** — Token is missing `manage_billing:enterprise` scope. Run: `gh auth refresh -s manage_billing:enterprise`

**Export stays "processing" >2 minutes** — Retry with a shorter date range (1–2 weeks). Very large enterprises may be slow.

**download_urls is empty or expired** — Pre-signed URLs expire in ~1 hour. Re-fetch the report object to get a fresh URL.

**CSV has unexpected columns** — Always check `reader.fieldnames` first and adapt column references accordingly. GitHub occasionally renames columns.
