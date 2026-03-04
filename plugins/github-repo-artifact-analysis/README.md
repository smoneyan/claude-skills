# github-repo-artifact-analysis

Deep-dive artifact storage analysis for a specific GitHub repository — find top consuming artifacts, identify cleanup opportunities, and optimize workflows.

## Usage

This skill is automatically triggered when you ask Claude about:
- Artifact storage in a specific repo
- Top artifacts consuming the most space
- Cleaning up a repo's Actions storage
- Artifact costs for a named repository

## What It Does

1. Fetches all artifacts for the repository (with pagination)
2. Ranks artifacts by total size and count
3. Shows age distribution (7d / 30d / 90d buckets)
4. Generates ready-to-run cleanup commands
5. Suggests workflow YAML optimizations to prevent future accumulation

## Requirements

- `gh` CLI installed and authenticated (`gh auth login`)
- Token scopes: `repo` + `actions:read`
