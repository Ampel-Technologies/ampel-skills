# Ampel Skills

The shared Claude Code skill library for the Ampel Technologies team.

## What are skills?

Skills are reusable workflows for Claude Code, triggered by slash commands like `/release v2.3.0`.
They automate repetitive tasks so the whole team benefits from them.

Skills are distributed as a **Claude Code plugin marketplace** — once you register the repo,
skills are installed automatically and stay up to date.

## Available skills

| Skill | Command | Description |
|-------|---------|-------------|
| `release` | `/release v1.2.3` | Create a tagged GitHub release with auto-generated release notes |
| `test-framework-dev` | `/test-framework-dev` | Development workflows for the Ampel test framework (FT-AMD) |

## First time setup

Add the following to your `~/.claude/settings.json`
(located at `C:\Users\<your-name>\.claude\settings.json` on Windows):

```json
{
  "extraKnownMarketplaces": {
    "ampel-skills": {
      "source": { "source": "github", "repo": "Ampel-Technologies/ampel-skills" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "release@ampel-skills": true,
    "test-framework-dev@ampel-skills": true
  }
}
```

Then restart Claude Code. The skills will be available immediately and will auto-update
whenever you restart.

> **Note:** If you already have other settings in `settings.json`, merge the above into your
> existing file rather than replacing it.

## Using the skills

Once installed, skills trigger automatically when relevant — or you can invoke them directly:

- **`/release v1.2.3`** — walks you through creating a GitHub release for the current repo
- **`/test-framework-dev`** — loads FT-AMD development context and workflows

## How to add a new skill

1. Clone this repo
2. Create a new folder under `skills/your-skill-name/`
3. Add a `SKILL.md` file following the structure of existing skills
4. Register it in `.claude-plugin/marketplace.json`
5. Open a PR — once merged, everyone gets it automatically on next restart

### Skill folder structure

```
skills/
  your-skill-name/
    SKILL.md          # Required: skill instructions and frontmatter
    references/       # Optional: reference docs loaded on demand
```

### SKILL.md frontmatter

```yaml
---
name: your-skill-name
description: >
  One paragraph describing what the skill does and when to trigger it.
  Be specific — this is what Claude uses to decide when to invoke the skill.
---
```

### Registering in the marketplace

After adding the skill folder, add an entry to `.claude-plugin/marketplace.json`:

```json
{
  "name": "your-skill-name",
  "source": "./skills/your-skill-name",
  "description": "Short description shown in the plugins UI"
}
```
