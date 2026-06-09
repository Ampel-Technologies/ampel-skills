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

### Step 1 - Find the skills installation directory

Search for the Claude skills directory dynamically with PowerShell:

```powershell
$skillsRoot = Get-ChildItem "$env:APPDATA\Claude\local-agent-mode-sessions\skills-plugin" `
    -Recurse -Directory -Filter "skill-creator" |
    Select-Object -First 1 |
    ForEach-Object { $_.Parent.FullName }

Write-Host "Skills directory: $skillsRoot"
```

This finds the folder containing `skill-creator` (a built-in skill that is always present),
which is the root where all skills live.

### Step 2 - Fetch available skills from GitHub

Call the GitHub API to list all skill folders in the repo:

```powershell
$headers = @{ Accept = "application/vnd.github+json" }
# Add auth header if token is available (avoids rate limits)
if ($env:GITHUB_TOKEN) {
    $headers["Authorization"] = "Bearer $env:GITHUB_TOKEN"
}

$response = Invoke-RestMethod `
    -Uri "https://api.github.com/repos/Ampel-Technologies/ampel-skills/contents/skills" `
    -Headers $headers

$available = $response | Where-Object { $_.type -eq "dir" } | Select-Object -ExpandProperty name
```

### Step 3 - Show what's available and what's already installed

Check which skills are already installed by listing subdirectories in `$skillsRoot`.
Then present a clear table to the user:

```
Available Ampel skills:

  [installed]  release        - Create tagged GitHub releases with auto-generated notes
  [installed]  ampel-install  - This skill (install/update Ampel skills)
  [available]  create-pr      - Create pull requests with auto-generated descriptions
  [available]  deploy         - Deploy to environment with release checklist

Which skills would you like to install/update? (Enter names separated by commas, or "all")
```

### Step 4 - Download and install selected skills

For each selected skill, recursively download all files from GitHub and write them to
`$skillsRoot\<skill-name>\`:

```powershell
function Install-AmpelSkill {
    param($skillName, $skillsRoot, $headers)

    $apiBase = "https://api.github.com/repos/Ampel-Technologies/ampel-skills/contents/skills/$skillName"
    $targetDir = Join-Path $skillsRoot $skillName

    function Download-Dir {
        param($apiUrl, $localPath)
        New-Item -ItemType Directory -Force $localPath | Out-Null
        $items = Invoke-RestMethod -Uri $apiUrl -Headers $headers
        foreach ($item in $items) {
            if ($item.type -eq "file") {
                $content = Invoke-RestMethod -Uri $item.download_url
                Set-Content -Path (Join-Path $localPath $item.name) -Value $content -Encoding UTF8
            } elseif ($item.type -eq "dir") {
                Download-Dir -apiUrl $item.url -localPath (Join-Path $localPath $item.name)
            }
        }
    }

    Download-Dir -apiUrl $apiBase -localPath $targetDir
    Write-Host "Installed: $skillName -> $targetDir"
}
```

Call this for each skill the user selected.

### Step 5 - Confirm and instruct restart

After installation:

```
Installed successfully:
  - release       -> C:\Users\...\skills\release\
  - create-pr     -> C:\Users\...\skills\create-pr\

Restart Claude Code (close and reopen) for the new skills to become active.
```

## Notes

- Skills are downloaded from the `main` branch of `Ampel-Technologies/ampel-skills`
- If a skill folder already exists it will be overwritten (safe update)
- No GITHUB_TOKEN is required (public repo), but having it set avoids GitHub API rate limits
- After install, a full restart of Claude Code is needed for skills to be picked up
