# smoneyan/claude-skills

A Claude Code plugin marketplace with skills for GitHub Actions analysis and DevOps productivity.

## Install

```
claude plugin add smoneyan/claude-skills
```

## Skills

### github-actions-storage

Analyze GitHub Actions artifact storage usage across an **entire organization** or from a billing CSV report.

**Triggers when you ask about:**
- Org-wide Actions storage costs
- Analyzing a billing report CSV
- Which repositories are using the most storage
- Reducing your GitHub Actions bill

### github-repo-artifact-analysis

Deep-dive artifact storage analysis for a **specific repository**.

**Triggers when you ask about:**
- Artifact storage in a specific repo
- Top artifacts by size
- Cleaning up a repo's Actions storage
- Artifact costs for a named repository

## Adding More Skills

Each skill lives in `plugins/<skill-name>/` with:
- `.claude-plugin/plugin.json` — plugin descriptor
- `skills/<skill-name>/SKILL.md` — skill content
- `README.md` — documentation

After adding a new plugin, register it in `.claude-plugin/marketplace.json`.
