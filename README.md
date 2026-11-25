# odoo-skills

Claude Code skills for Odoo development.

| Skill | What it does |
|-------|--------------|
| [`pr-review`](pr-review/SKILL.md) | Reviews a commit, branch, or PR diff (code and commit message) thoroughly enough to merge after one pass. |
| [`spec-grill`](spec-grill/SKILL.md) | Tests a vague feature spec before development: five isolated reviewers report to a blind judge. |

## Install

Inside Claude Code:

```
/plugin install odoo-skills --marketplace rushil-b-patel/odoo-skills
```

On Claude Code older than 2.1.275, use two steps:

```
/plugin marketplace add rushil-b-patel/odoo-skills
/plugin install odoo-skills@odoo-skills
```

The skills load automatically when your request matches them, or you can call them directly with `/odoo-skills:pr-review` or `/odoo-skills:spec-grill`.

### Updates

**Automatic (recommended):** run `/plugin`, open **Marketplaces**, select `odoo-skills`, and choose **Enable auto-update**. Claude Code then pulls new versions each time it starts. Third-party marketplaces have auto-update turned off by default.

**Manual:**

```
/plugin marketplace update odoo-skills
/plugin update odoo-skills@odoo-skills
```

Restart Claude Code after updating. To uninstall, run `/plugin uninstall odoo-skills`.

### Manual install (no plugin system)

```bash
git clone https://github.com/rushil-b-patel/odoo-skills.git
mkdir -p ~/.claude/skills
ln -s "$PWD/odoo-skills/pr-review" "$PWD/odoo-skills/spec-grill" ~/.claude/skills/
```

Because these are symlinks, `git pull` updates the skills. To install for one project only, link the skills into `<project>/.claude/skills/` instead.
