---
name: release
description: >
  Automates creating GitHub releases from the current git repo. Use this skill whenever
  the user wants to create a release, publish a version, tag a commit for release, generate
  release notes, or publish to GitHub releases -- even if they just say "make a release",
  "release v2.3.0", "tag and publish this version", or "create a new release". Invoke it
  with a version number like `/release v2.2.5`. The skill finds commits since the last tag,
  categorizes them into release notes, confirms with the user, then creates the git tag and
  publishes the GitHub release automatically using the GITHUB_TOKEN environment variable.
---

# Release Skill

This skill walks you through creating a tagged GitHub release from the current repo. It works entirely via git commands and the GitHub REST API -- no `gh` CLI needed.

## Prerequisites

- `GITHUB_TOKEN` environment variable must be set (with `repo` scope)
- You must be in a git repository with a GitHub remote

## Workflow

### Step 1 - Parse the version

The user invokes this skill with a version number, e.g.:
```
/release v2.3.0
```

Extract the version string (include the `v` prefix if provided). If no version was given, ask the user for one before proceeding.

### Step 2 - Gather repo info

Run these commands to understand the current state:

```powershell
# Get the GitHub owner/repo from the remote URL
git remote get-url origin

# Get the last tag (to find commits since then)
git describe --tags --abbrev=0

# Get commits since the last tag
git log <last-tag>..HEAD --oneline

# Get the latest commit hash (this is where the tag will go)
git rev-parse HEAD
```

Parse the remote URL to extract `owner/repo`. It may be SSH (`git@github.com:owner/repo.git`) or HTTPS (`https://github.com/owner/repo.git`).

### Step 3 - Generate release notes

Look at the commit messages since the last tag and categorize them into sections. Use your judgment based on the commit message wording:

- **New Features** -- commits mentioning "add", "new", "implement", "introduce", "support"
- **Bug Fixes** -- commits mentioning "fix", "bug", "patch", "correct", "resolve"
- **Improvements** -- commits mentioning "update", "improve", "refactor", "optimize", "change", "publish", "config"

Format the release notes as:

```markdown
## v2.3.0

### New Features
- Description of feature (from commit message, cleaned up)

### Bug Fixes
- Description of fix

### Improvements
- Description of improvement
```

Only include sections that have entries. Clean up commit messages slightly for readability (capitalize first letter, remove redundant words) but stay faithful to what was done.

### Step 4 - Show proposal and ask for confirmation

Present a clear summary to the user:

```
Ready to create release:

  Version:    v2.3.0
  Tag target: <full commit hash> (HEAD)
  Last tag:   v2.2.4

Release notes:
---
<formatted release notes>
---

Proceed? (yes/no)
```

Wait for the user to confirm before doing anything. If they want to edit the release notes or change anything, incorporate their feedback and show the updated proposal again.

### Step 5 - Create the tag and push it

Once confirmed:

```powershell
git tag <version> HEAD
git push origin <version>
```

If the tag already exists locally, warn the user and stop -- do not overwrite existing tags without explicit confirmation.

### Step 6 - Publish the GitHub release via API

Call the GitHub API using PowerShell:

```powershell
$body = @{
    tag_name   = "<version>"
    name       = "<version>"
    body       = "<release notes as a string with \n for newlines>"
    draft      = $false
    prerelease = $false
} | ConvertTo-Json

$response = Invoke-RestMethod `
    -Uri "https://api.github.com/repos/<owner>/<repo>/releases" `
    -Method Post `
    -Headers @{
        Authorization = "Bearer $env:GITHUB_TOKEN"
        Accept        = "application/vnd.github+json"
    } `
    -Body $body `
    -ContentType "application/json"

Write-Host $response.html_url
```

### Step 7 - Report success

Return the release URL to the user:

```
Release v2.3.0 published!
https://github.com/owner/repo/releases/tag/v2.3.0
```

## Error handling

- **No GITHUB_TOKEN**: Tell the user to set it as a Windows environment variable (Start -> "Environment Variables" -> New user variable), then restart Claude Code.
- **401 Unauthorized**: Token is set but invalid or missing `repo` scope. Direct user to https://github.com/settings/tokens to generate a new one.
- **Tag already exists**: Warn the user. Do not force-push or overwrite without explicit instruction.
- **No commits since last tag**: Let the user know -- they may still want to create the release on the existing HEAD.
- **No previous tag found**: Use the full commit history for release notes, or ask the user what range to use.
