---
name: github-repo-artifact-analysis
description: Analyze GitHub Actions artifact storage for a specific repository — find top consuming artifacts, identify cleanup opportunities, and optimize workflows. Trigger when user asks about artifact storage in a specific repo, wants to see top artifacts, asks which artifacts are consuming the most space, wants to clean up a repo's Actions storage, or asks about artifact costs for a named repository.
---

# GitHub Repository Artifact Analysis

When this skill is triggered, perform a live analysis using `gh` CLI commands. Do not just provide documentation — execute the analysis and present real findings.

## Step 1: Identify the Repository

Extract `{org}` and `{repo}` from the user's request. If ambiguous, ask for clarification before proceeding.

## Step 2: Fetch All Artifacts

```bash
# Fetch all artifacts with pagination (handles repos with many artifacts)
gh api "repos/{org}/{repo}/actions/artifacts?per_page=100" --paginate \
  --jq '.artifacts[] | {id: .id, name: .name, size_mb: (.size_in_bytes / 1048576 | . * 10 | round / 10), created_at: .created_at, expired: .expired, workflow_run_id: .workflow_run.id}'
```

If the repository is large (>1000 artifacts), warn the user and offer to limit to the most recent 500.

## Step 3: Aggregate and Rank

After fetching, compute:

1. **Top artifacts by total size** — group by artifact `name`, sum `size_mb`, count occurrences
2. **Top artifacts by count** — same grouping, sorted by count
3. **Age distribution** — categorize into: <7 days, 7–30 days, 30–90 days, >90 days
4. **Total storage used** — sum all `size_mb`
5. **Expired artifacts** — filter `expired: true`, sum size

Use this jq pipeline to get a ranked summary:
```bash
gh api "repos/{org}/{repo}/actions/artifacts?per_page=100" --paginate \
  --jq '[.artifacts[] | {name: .name, size_mb: (.size_in_bytes / 1048576), created_at: .created_at, expired: .expired}] | group_by(.name) | map({name: .[0].name, total_mb: (map(.size_mb) | add | . * 10 | round / 10), count: length, expired_count: (map(select(.expired)) | length)}) | sort_by(-.total_mb) | .[0:20]'
```

## Step 4: Present Findings

Format the output as:

```
## Artifact Storage Analysis: {org}/{repo}

**Total artifacts**: N
**Total storage**: X.X GB
**Expired (not yet GC'd)**: X.X GB

### Top Consuming Artifacts (by total size)

| Rank | Artifact Name | Total Size | Count | Avg Size | Safe to Clean |
|------|--------------|------------|-------|----------|---------------|
| 1    | name         | X.X GB     | N     | X MB     | ✅/⚠️/❌      |

### Age Distribution
- < 7 days: N artifacts (X GB)
- 7–30 days: N artifacts (X GB)
- 30–90 days: N artifacts (X GB)
- > 90 days: N artifacts (X GB) ← Cleanup priority
```

## Step 5: Cleanup Recommendations

### Immediate Actions (Safe)
Generate ready-to-run commands for:
- All artifacts >90 days old
- All expired artifacts
- Test/log artifacts >7 days

```bash
# Delete all artifacts older than 90 days
gh api "repos/{org}/{repo}/actions/artifacts?per_page=100" --paginate \
  --jq '.artifacts[] | select(.created_at < "'$(date -v-90d +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -d '90 days ago' +%Y-%m-%dT%H:%M:%SZ)'") | .id' \
  | xargs -I{} gh api -X DELETE "repos/{org}/{repo}/actions/artifacts/{}"
```

### Workflow Optimizations
Based on the artifact names found, suggest specific workflow changes:

**If test/playwright/cypress artifacts are large:**
```yaml
- uses: actions/upload-artifact@v4
  if: failure() || github.ref == 'refs/heads/main'
  with:
    name: test-results-${{ github.run_id }}
    path: test-results/
    retention-days: 3
```

**If build artifacts are accumulating:**
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-${{ github.sha }}
    path: dist/
    retention-days: 30
```

**If matrix jobs multiply artifacts:**
```yaml
# Upload from one job only
- uses: actions/upload-artifact@v4
  if: matrix.node-version == '20'
  with:
    name: shared-artifact
```

## Cleanup Safety Guide

| Artifact Type | Age | Safety | Action |
|--------------|-----|--------|--------|
| test-results, playwright-report, coverage | >7 days | ✅ Safe | Delete |
| logs, debug | >14 days | ✅ Safe | Delete |
| build, dist | >30 days | ⚠️ Review | Verify not in use |
| release, package | >90 days | ⚠️ Review | Check with team |
| deploy, production | Any | ❌ Risk | Team approval required |

## Follow-up Options

After presenting findings, offer the user:
1. **Generate delete script** — produce a shell script with all safe cleanup commands
2. **Analyze specific artifact type** — drill down into a particular artifact group
3. **Show workflow file** — examine the workflow generating large artifacts
4. **Set up retention policy** — show the exact YAML to add to their workflows
