---
name: ampel-install
description: >
  Installs Ampel team skills from the ampel-skills GitHub repository into Claude Code.
  Use this skill whenever the user wants to install a team skill, browse available Ampel
  skills, update an existing skill, or asks "what skills do we have?". Invoke with
  `/ampel-install` to see the full list and pick which ones to install. Also handles
  updating already-installed skills to their latest version from GitHub.
---

# Ampel Install Skill

This skill browses the `Ampel-Technologies/ampel-skills` GitHub repository and installs
selected skills directly into Claude Code -- no manual file copying needed.

## Workflow

### Step 1 - Find all skills session directories and their manifests

Skills in Claude Code live under session-specific directories. We must update ALL of them
so skills appear regardless of which session is active:

```powershell
# Find all manifest.json files under the skills-plugin directory
$manifests = Get-ChildItem "$env:APPDATA\Claude\local-agent-mode-sessions\skills-plugin" `
    -Recurse -Filter "manifest.json" | Select-Object -ExpandProperty FullName

# The skills folder is always a sibling of manifest.json
$skillsDirs = $manifests | ForEach-Object { Join-Path (Split-Path $_) "skills" }

Write-Host "Found $($manifests.Count) session(s) to update"
```

### Step 2 - Fetch available skills from GitHub

```powershell
$headers = @{ Accept = "application/vnd.github+json" }
if ($env:GITHUB_TOKEN) { $headers["Authorization"] = "Bearer $env:GITHUB_TOKEN" }

$response = Invoke-RestMethod `
    -Uri "https://api.github.com/repos/Ampel-Technologies/ampel-skills/contents/skills" `
    -Headers $headers

$available = $response | Where-Object { $_.type -eq "dir" } | Select-Object name, url
```

### Step 3 - Show what's available vs installed

Check which skills exist in the primary skills directory, then show the user:

```
Available Ampel skills:

  [installed]  release             - Create tagged GitHub releases with auto-generated notes
  [installed]  ampel-install       - This skill (install/update Ampel skills)
  [available]  test-framework-dev  - FT-AMD test framework development workflows

Which skills would you like to install/update? (names separated by commas, or "all")
```

### Step 4 - Download skill files

For each selected skill, recursively download all files from GitHub:

```powershell
function Download-SkillDir {
    param($apiUrl, $localPath, $headers)
    New-Item -ItemType Directory -Force $localPath | Out-Null
    $items = Invoke-RestMethod -Uri $apiUrl -Headers $headers
    foreach ($item in $items) {
        if ($item.type -eq "file") {
            $content = Invoke-RestMethod -Uri $item.download_url
            Set-Content -Path (Join-Path $localPath $item.name) -Value $content -Encoding UTF8
        } elseif ($item.type -eq "dir") {
            Download-SkillDir -apiUrl $item.url -localPath (Join-Path $localPath $item.name) -headers $headers
        }
    }
}

# Install into EVERY skills directory found in Step 1
foreach ($skillsDir in $skillsDirs) {
    $targetDir = Join-Path $skillsDir $skillName
    Download-SkillDir -apiUrl $skillApiUrl -localPath $targetDir -headers $headers
}
```

### Step 5 - Read the skill's description from its SKILL.md

Parse the `description` field from the YAML frontmatter of the downloaded SKILL.md.
This is needed to register it in the manifest.

```powershell
$skillMdPath = Join-Path $primarySkillsDir "$skillName\SKILL.md"
$content = Get-Content $skillMdPath -Raw

# Extract description between frontmatter markers
if ($content -match "(?s)---.*?description: >?\s*\n([\s\S]+?)(?:---|\w+:)") {
    $description = ($matches[1] -replace "\s+", " ").Trim()
}
```

### Step 6 - Register the skill in ALL manifest.json files

This is critical -- skills only appear in Claude Code if they are listed in the manifest.
Update every manifest found in Step 1:

```powershell
foreach ($manifestPath in $manifests) {
    $manifest = Get-Content $manifestPath -Raw | ConvertFrom-Json
    $existingIds = $manifest.skills | ForEach-Object { $_.skillId }

    if ($existingIds -notcontains $skillName) {
        $manifest.skills += [PSCustomObject]@{
            skillId     = $skillName
            name        = $skillName
            description = $description
            creatorType = "user"
            updatedAt   = $null
            enabled     = $true
        }
    } else {
        # Update description in case it changed
        $existing = $manifest.skills | Where-Object { $_.skillId -eq $skillName }
        $existing.description = $description
    }

    $manifest | ConvertTo-Json -Depth 10 | Set-Content $manifestPath -Encoding UTF8
}
```

### Step 7 - Confirm and instruct reload

```
Installed successfully:
  - test-framework-dev

Type /reload-skills to activate them immediately (no restart needed).
```

## Notes

- Skills must be registered in manifest.json to appear -- copying files alone is not enough
- All session manifests are updated so skills persist across session changes
- No GITHUB_TOKEN required (public repo), but it avoids GitHub API rate limits
- Use /reload-skills after install instead of restarting Claude Code
