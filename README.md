# Ampel Skills

The shared Claude Code skill library for the Ampel Technologies team.

## What are skills?

Skills are reusable workflows for Claude Code, triggered by slash commands like `/release v2.3.0`.
They automate repetitive tasks so the whole team benefits from them.

## Available skills

| Skill | Command | Description |
|-------|---------|-------------|
| `release` | `/release v1.2.3` | Create a tagged GitHub release with auto-generated release notes |
| `ampel-install` | `/ampel-install` | Browse and install skills from this repo |
| `test-framework-dev` | `/test-framework-dev` | Development workflows for the Ampel test framework (FT-AMD) |

## How to install skills

### First time setup

1. Open Claude Code
2. Type `/ampel-install`
3. Pick which skills to install
4. Restart Claude Code

### Updating skills

Run `/ampel-install` again and select the skills you want to update. It will overwrite
the local version with the latest from this repo.

## How to add a new skill

1. Clone this repo
2. Create a new folder under `skills/your-skill-name/`
3. Add a `SKILL.md` file following the structure of existing skills
4. Open a PR -- once merged, the skill is available to the whole team via `/ampel-install`

## Skill folder structure

```
skills/
  your-skill-name/
    SKILL.md          # Required: skill instructions and frontmatter
    scripts/          # Optional: helper scripts the skill uses
    references/       # Optional: reference docs loaded on demand
```

The `SKILL.md` frontmatter must include:
```yaml
---
name: your-skill-name
description: >
  One paragraph describing what the skill does and when to trigger it.
---
```
