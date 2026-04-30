---
name: skill-publisher
description: "Generic publishing tool for Claude Code skills. Validates, packages, and publishes to ClawHub, Hermes Agent, and anthropics/skills with bilingual README support."
version: 1.0.0
author: Claude Code
license: MIT
dependencies: []
metadata:
  hermes:
    tags: [Skill, Publishing, ClawHub, Hermes, Claude, Workflow, Automation]
    related_skills: [ascii-excalidraw]
---

# Skill Publisher

A generic publishing workflow for any Claude Code skill. Package, validate, and publish to **ClawHub** (OpenClaw), **Hermes Agent**, and **anthropics/skills** (Claude Code).

## When to Use

- User wants to publish a skill they've built to one or more marketplaces
- User needs a bilingual (EN/ZH) README for their skill
- User wants to validate SKILL.md format before publishing
- User needs help with `clawhub publish` or GitHub PR submission

## Prerequisites

Before using this skill, check for the following in the environment (or ask the user):

### Authentication

| Token | Purpose | Env Var |
|---|---|---|
| GitHub PAT | GitHub API calls | `GITHUB_TOKEN` |
| ClawHub token | `clawhub publish` | `CLAWHUB_TOKEN` |

Tokens can be in env vars, a `.env` file, or provided interactively.

### Proxy (if needed)

If `api.github.com` is blocked, set:

```bash
export https_proxy=http://<proxy-ip>:<port>
```

Per-command: `curl -x http://<proxy>`, `git -c http.proxy=http://<proxy>`, `https_proxy=http://<proxy> clawhub publish`

**Note:** SSH git (`git@github.com:...`) works without proxy on port 22.

## Workflow

### Step 1: Validate Source Skill

```bash
python3 ~/.skills/skill-publisher/scripts/validate_skill.py <skill-path>
```

Checks: SKILL.md exists, valid YAML frontmatter, required fields (`name`, `description`), name format (lowercase, hyphens), description <= 1024 chars, no hardcoded secrets, file size <= 100,000 chars.

### Step 2: Generate Publish Package

```bash
python3 ~/.skills/skill-publisher/scripts/generate_package.py \
  --source ~/.skills/<skill-name> \
  --output /tmp/publish-<skill-name> \
  --platforms clawhub hermes anthropics
```

Produces:

```
/tmp/publish-<skill-name>/
├── SKILL.md            # Copied from source
├── README.md           # Bilingual (EN/ZH) README with examples
├── examples/           # Copied from source skill
└── .clawhub.json       # ClawHub manifest
```

### Step 3: Platform-Specific Publishing

#### ClawHub (OpenClaw)

```bash
cd /tmp/publish-<skill-name>
clawhub publish .
```

If CLI unavailable: install `npm i -g clawhub` or upload via clawhub.ai.

#### Hermes Agent (GitHub PR)

Target: `NousResearch/hermes-agent` → `skills/creative/<skill-name>/SKILL.md`

1. Fork repo on GitHub
2. Clone fork, copy SKILL.md to `skills/creative/<skill-name>/`
3. Commit, push, create PR

#### anthropics/skills (GitHub PR)

Target: `anthropics/skills` → `skills/<skill-name>/SKILL.md`

Same process as Hermes, different repo.

### Step 4: Verify

```bash
# ClawHub
curl -s "https://clawhub.ai/api/v1/skills?slug=<skill-name>"
# Hermes / anthropics — check raw GitHub URLs
```

## Helper Scripts

| Script | Purpose |
|---|---|
| `scripts/validate_skill.py` | Validate SKILL.md format |
| `scripts/generate_package.py` | Generate publish package with bilingual README |
| `scripts/gen_clawhub_manifest.py` | Generate .clawhub.json |
| `scripts/create_pr.py` | Create GitHub PR via API |

## Common Pitfalls

1. **SKILL.md first bytes must be `---`** — no BOM, no blank line
2. **ClawHub rejects hardcoded secrets** — scan scripts/ before publishing
3. **Hermes requires metadata** — `metadata.hermes.tags` and `related_skills`
4. **Network issues** — if api.github.com unreachable, use SSH push to fork or manual PR via UI
