# DevSecOps Skills for Claude Code

Claude Code skills for GitHub and security engineering productivity.

## Install

Install all plugins:
```
claude plugin add smoneyan/claude-skills
```

Install a specific plugin:
```
claude plugin add smoneyan/claude-skills --plugin github-skills
claude plugin add smoneyan/claude-skills --plugin security-skills
```

---

## Plugins

### `github-skills`

GitHub Actions storage and artifact analysis — identify top consuming repositories, analyze billing reports, deep-dive artifact cleanup, and optimize workflow retention policies.

| Skill | Description |
|-------|-------------|
| `github-actions-storage` | Analyze artifact storage across an entire org or billing CSV — rank repos by usage, estimate costs, build cleanup plans |
| `github-repo-artifact-analysis` | Deep-dive analysis for a specific repo — top artifacts, age distribution, ready-to-run cleanup commands |
| `github-billing-usage-reports` | Export and analyze GitHub Enterprise billing usage — spending by product, repo, org, or user |

### `security-skills`

Security scanning and vulnerability management for GitHub Enterprise — assess CVE impact across all organizations and triage security advisories.

| Skill | Description |
|-------|-------------|
| `github-cve-scanner` | Given a CVE URL or ID, fetch details from NVD, scan all GHE orgs for the vulnerable package in dependency files, and report affected repos |

**Example trigger:**
> *"Check CVE-2021-44228 against our GitHub Enterprise at github.mycompany.com, enterprise slug acme-corp"*

---

## Repository Structure

```
claude-skills/
├── .claude-plugin/
│   └── marketplace.json         # marketplace descriptor
├── skills/
│   ├── github-actions-storage/
│   ├── github-billing-usage-reports/
│   ├── github-repo-artifact-analysis/
│   └── github-cve-scanner/
│       ├── SKILL.md
│       ├── scripts/
│       │   ├── parse_cve.py     # NVD API parser + package ranker
│       │   └── cve_scanner.py   # multi-org gh code search with rate limiting
│       └── references/
│           └── ecosystem-patterns.md
└── README.md
```

## Adding More Skills

1. Create `skills/<skill-name>/SKILL.md`
2. Add the path to the relevant plugin in `.claude-plugin/marketplace.json`
