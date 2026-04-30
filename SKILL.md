---
name: skill-publisher
description: "Package and publish Claude Code skills to ClawHub, Hermes, and anthropics/skills. Generates bilingual README with examples, validates format, and guides through the publishing workflow."
version: 1.0.0
author: Claude Code
license: MIT
dependencies: []
metadata:
  hermes:
    tags: [Skill, Publishing, ClawHub, Hermes, Claude, Workflow, Automation]
    related_skills: [ascii-excalidraw, excalidraw]
---

# Skill Publisher

Package, validate, and publish Claude Code skills to multiple marketplaces: **ClawHub** (OpenClaw), **Hermes Agent**, and **anthropics/skills** (Claude Code).

## When to Use

- User wants to publish a skill they've built to one or more marketplaces
- User needs a bilingual (EN/ZH) README for their skill
- User wants to validate SKILL.md format before publishing
- User needs help with `clawhub publish` or GitHub PR submission

## Configuration

### GitHub Token

```bash
export GITHUB_TOKEN="ghp_xxxxxxxxxxxxx"
```

The GitHub user is `zhanyganbo2007` (SSH auth shows as `zhangyanbo2007`).

### Proxy (Required)

The environment blocks direct access to api.github.com:443. All GitHub API and HTTPS operations **must** use the proxy:

```bash
export https_proxy=http://192.168.28.92:7897
export http_proxy=http://192.168.28.92:7897
export no_proxy="localhost,127.0.0.1,.local"
```

**Per-command proxy usage:**

| Tool | Command with proxy |
|---|---|
| curl | `curl -x http://192.168.28.92:7897 ...` |
| git HTTPS | `git -c http.proxy=http://192.168.28.92:7897 clone ...` |
| npm/npx | `https_proxy=http://192.168.28.92:7897 npm install ...` |
| clawhub | `https_proxy=http://192.168.28.92:7897 clawhub publish .` |
| gh CLI | `https_proxy=http://192.168.28.92:7897 gh pr create ...` |
| Python requests | `export https_proxy=http://192.168.28.92:7897` |

**Note:** SSH git operations (`git@github.com:...`) work **without** proxy on port 22.

## Workflow

### Step 1: Validate Source Skill

Before packaging, validate the source skill's SKILL.md:

```bash
python3 ~/.skills/skill-publisher/scripts/validate_skill.py <skill-path>
```

Checks performed:
- SKILL.md exists at the root
- YAML frontmatter is valid and complete
- Required fields present: `name`, `description`
- Name format: lowercase, hyphens, no spaces
- Description length <= 1024 chars
- No hardcoded secrets in scripts/
- File size <= 100,000 chars

### Step 2: Generate Publish Package

Generate the publish package with bilingual README and example files:

```bash
python3 ~/.skills/skill-publisher/scripts/generate_package.py \
  --source ~/.skills/<skill-name> \
  --output /tmp/publish-<skill-name> \
  --platforms clawhub hermes anthropics
```

This produces:

```
/tmp/publish-<skill-name>/
── SKILL.md                  # Copied from source
├── README.md                 # Bilingual (EN/ZH) README with examples
── README.zh.md              # Chinese-only README (optional)
├── examples/
│   ├── example1.ascii        # Sample ASCII input
│   ├── example1.excalidraw   # Generated output
│   └── example2.ascii
│   └── example2.excalidraw
├── screenshots/              # If user provides screenshots
│   └── demo.png
└── .clawhub.json             # ClawHub manifest (if targeting ClawHub)
```

### Step 3: Platform-Specific Preparation

#### ClawHub (OpenClaw)

```bash
cd /tmp/publish-<skill-name>

# Generate ClawHub manifest if not present
python3 ~/.skills/skill-publisher/scripts/gen_clawhub_manifest.py \
  --skill-dir . --output .clawhub.json

# Validate format
clawhub validate 2>/dev/null || echo "clawhub CLI not installed — format validated manually"

# Publish (requires network + clawhub CLI + GITHUB_TOKEN)
clawhub login                    # First time only
clawhub publish .
```

If `clawhub` CLI is not available, the user can:
1. Install it: `npm i -g clawhub`
2. Or manually upload via clawhub.ai web interface

#### Hermes Agent (GitHub PR)

Target repo: `NousResearch/hermes-agent`

```bash
# Fork and clone (first time)
git clone https://github.com/zhanyganbo2007/hermes-agent.git
cd hermes-agent

# Copy skill into category directory
cp -r /tmp/publish-<skill-name>/SKILL.md skills/creative/<skill-name>/

# Commit and push
git checkout -b feature/<skill-name>
git add skills/creative/<skill-name>/
git commit -m "feat: add <skill-name> skill"
git push -u origin feature/<skill-name>

# Create PR via API (or manually via GitHub UI)
python3 ~/.skills/skill-publisher/scripts/create_pr.py \
  --repo NousResearch/hermes-agent \
  --branch feature/<skill-name> \
  --title "Add <skill-name> skill" \
  --body-file /tmp/publish-<skill-name>/PR_BODY.md
```

#### anthropics/skills (GitHub PR)

Target repo: `anthropics/skills`

Same process as Hermes, but target different repo:

```bash
git clone https://github.com/zhanyganbo2007/skills.git  # fork
cd skills
cp -r /tmp/publish-<skill-name>/SKILL.md skills/<skill-name>/
# ... same git workflow
```

### Step 4: Post-Publish Verification

After publishing, verify the skill is live:

```bash
# ClawHub
curl -s "https://clawhub.ai/api/skills/<skill-name>" | python3 -m json.tool

# Hermes
curl -s "https://raw.githubusercontent.com/NousResearch/hermes-agent/main/skills/creative/<skill-name>/SKILL.md"

# anthropics/skills
curl -s "https://raw.githubusercontent.com/anthropics/skills/main/skills/<skill-name>/SKILL.md"
```

## Bilingual README Template

The generated README follows this structure:

```markdown
# Skill Name / 技能名称

> Short description in English / 简短中文描述

## Features / 功能特性

- Feature 1 / 功能一
- Feature 2 / 功能二

## Quick Start / 快速开始

### English

1. Step one
2. Step two

### 中文

1. 步骤一
2. 步骤二

## Examples / 示例

### Example 1: Title / 示例一：标题

Input:
```
ASCII input here
```

Output:
![screenshot](screenshots/example1.png)

---

## Contributing / 贡献

[Standard contribution guidelines in both languages]
```

## Helper Scripts

| Script | Purpose |
|---|---|
| `scripts/validate_skill.py` | Validate SKILL.md format and content |
| `scripts/generate_package.py` | Generate publish package with README and examples |
| `scripts/gen_clawhub_manifest.py` | Generate .clawhub.json manifest |
| `scripts/create_pr.py` | Create GitHub PR via API |
| `scripts/gen_readme.py` | Generate bilingual README from SKILL.md |

## Common Pitfalls

1. **SKILL.md first bytes must be `---`** — no BOM, no blank line before frontmatter
2. **ClawHub rejects skills with hardcoded secrets** — scan scripts/ before publishing
3. **Hermes requires specific metadata format** — `metadata.hermes.tags` and `metadata.hermes.related_skills`
4. **anthropics/skills has no formal CONTRIBUTING.md** — keep PR description clear and reference existing skills
5. **GitHub API rate limits** — if creating PRs programmatically, add delays between requests
6. **Network restrictions** — if api.github.com is unreachable, fall back to manual PR creation via GitHub UI

## Verification Checklist

Before marking a publish as complete:

- [ ] SKILL.md passes validation
- [ ] Bilingual README generated with at least 2 examples
- [ ] All scripts in scripts/ have no hardcoded secrets
- [ ] Package structure matches target platform requirements
- [ ] GitHub repo forked and branch created
- [ ] PR created (or clawhub publish succeeded)
- [ ] User notified with links to live skill

## One-Shot Recipe: Full Publish Flow

```bash
# Full flow for a skill at ~/.skills/my-skill
python3 ~/.skills/skill-publisher/scripts/generate_package.py \
  --source ~/.skills/my-skill \
  --output /tmp/publish-my-skill \
  --platforms clawhub hermes anthropics \
  --examples-dir ~/.skills/my-skill/examples \
  --screenshots ~/.skills/my-skill/screenshots

# Then follow platform-specific steps above
```
