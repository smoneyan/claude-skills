# smoneyan/claude-skills

Claude Code skills for GitHub Actions analysis and DevOps productivity.

## Install

```
claude plugin add smoneyan/claude-skills
```

## Plugins

### github-skills

A collection of GitHub Actions analysis skills.

| Skill | Description |
|-------|-------------|
| `github-actions-storage` | Analyze artifact storage across an entire org or billing CSV — rank repos by usage, estimate costs, build cleanup plans |
| `github-repo-artifact-analysis` | Deep-dive analysis for a specific repo — top artifacts, age distribution, ready-to-run cleanup commands |

## Repository Structure

```
claude-skills/
├── .claude-plugin/
│   └── marketplace.json     # marketplace descriptor
├── skills/
│   ├── github-actions-storage/
│   │   └── SKILL.md
│   └── github-repo-artifact-analysis/
│       └── SKILL.md
└── README.md
```

## Adding More Skills

1. Create `skills/<skill-name>/SKILL.md`
2. Add the path to the `skills` array in `.claude-plugin/marketplace.json`
