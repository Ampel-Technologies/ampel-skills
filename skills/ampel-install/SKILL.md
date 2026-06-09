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

### Step 1 - Find ALL session directories (current and future-proof)

Skills live inside `local-agent-mode-sessions\skills-plugin\<outer-uuid>\<inner-uuid>\`.
New outer sessions are created every time Claude Code starts a fresh context -- we must
update ALL of them so skills survive restarts and new windows.

```powershell
$pluginBase = "$env:APPDATA\Claude\local-agent-mode-sessions\skills-plugin"

# Find every manifest.json recursively -- covers all current and future outer sessions
$manifests = Get-ChildItem $pluginBase -Recurse -Filter "manifest.json" |
    Select-Object -ExpandProperty FullName

# Derive the skills folder from each manifest path
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

Check which skills exist in the first skills directory, then show the user:

```
Available Ampel skills:

  [installed]  release             - Create tagged GitHub releases with auto-generated notes
  [installed]  ampel-install       - This skill (install/update Ampel skills)
  [available]  test-framework-dev  - FT-AMD test framework development workflows

Which skills would you like to install/update? (names separated by commas, or "all")
```

### Step 4 - Download skill files into ALL sessions

For each selected skill, recursively download all files from GitHub into every session:

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

foreach ($skillsDir in $skillsDirs) {
    $targetDir = Join-Path $skillsDir $skillName
    Download-SkillDir -apiUrl $skillApiUrl -localPath $targetDir -headers $headers
}
```

### Step 5 - Read the skill's description from its SKILL.md

Parse the `description` field from the YAML frontmatter of the downloaded SKILL.md.

```powershell
$primarySkillsDir = $skillsDirs[0]
$skillMdPath = Join-Path $primarySkillsDir "$skillName\SKILL.md"
$content = Get-Content $skillMdPath -Encoding UTF8 | Out-String

if ($content -match "(?s)---.*?description: >?\s*\n([\s\S]+?)(?:---|\w+:)") {
    $description = ($matches[1] -replace "\s+", " ").Trim()
}
```

### Step 6 - Register the skill in ALL manifest.json files

```powershell
foreach ($manifestPath in $manifests) {
    $manifest = Get-Content $manifestPath -Encoding UTF8 | ConvertFrom-Json
    $existingIds = $manifest.skills | ForEach-Object { $_.skillId }

    if ($existingIds -notcontains $skillName) {
        $newEntry = [PSCustomObject]@{
            skillId     = $skillName
            name        = $skillName
            description = $description
            creatorType = "user"
            updatedAt   = $null
            enabled     = $true
        }
        # Build a new array (+= on PSCustomObject arrays can lose type info)
        $manifest.skills = @($manifest.skills) + $newEntry
    } else {
        $existing = $manifest.skills | Where-Object { $_.skillId -eq $skillName }
        $existing.description = $description
    }

    $manifest | ConvertTo-Json -Depth 10 | Set-Content $manifestPath -Encoding UTF8
}
```

### Step 7 - Handle new sessions created after install

New Claude Code windows create a fresh outer session UUID that won't have user skills.
After installing, remind the user to re-run `/ampel-install` or `/reload-skills` whenever
they open a brand-new Claude Code window for the first time. To check if skills are missing:

```powershell
# Quick check: how many outer sessions exist, and how many have our skills?
$allManifests = Get-ChildItem "$env:APPDATA\Claude\local-agent-mode-sessions\skills-plugin" `
    -Recurse -Filter "manifest.json"
foreach ($m in $allManifests) {
    $skills = (Get-Content $m.FullName -Encoding UTF8 | ConvertFrom-Json).skills
    $userSkills = $skills | Where-Object { $_.creatorType -eq "user" }
    Write-Host "$($m.FullName): $($userSkills.Count) user skill(s)"
}
```

### Step 8 - Confirm and instruct reload

```
Installed successfully:
  - test-framework-dev

Run /reload-skills to activate immediately.
If skills don't appear after reload, restart Claude Code -- new sessions require a full restart.
```

## Notes

- Skills must be registered in manifest.json to appear -- copying files alone is not enough
- All session manifests are updated so skills persist across session changes
- **New Claude Code windows** create new outer session UUIDs. Run `/ampel-install` once per
  new window to re-populate it, or restart Claude Code after install so it picks up all sessions
- No GITHUB_TOKEN required (public repo), but it avoids GitHub API rate limits
