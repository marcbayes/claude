# Build the AI Launcher: instructions for Claude Code

> **For the human:** do this *after* the first guide (`AI-Terminal-Setup-Guide.md`)
> is finished and Claude Code works for you. When you're ready to start
> connecting apps, open Claude Code, drop this file in, and say:
> **"Follow the instructions in this file."**
>
> It builds a double-click launcher. You pick Claude or Codex, pick Work or
> Personal, and then chat, add a connection, or see your connections. It also
> creates a second guide that Claude uses to walk you through adding each new
> app, so you can come back to it any time.

---

## Instructions for Claude (the agent)

You are building a small launcher for a non-technical user. It has three
parts, all created in `%USERPROFILE%\AI Sessions` (the folder from the first
setup guide):

| File | Purpose |
|---|---|
| `AI-Launcher.bat` | The double-click file. Opens Windows Terminal and runs the menu. |
| `launcher.ps1` | The menu: choose assistant, choose folder, then chat / add connection / list connections. |
| `Add-Connection-Guide.md` | Instructions you (or a later Claude session) follow to add one MCP connection. |

### Ground rules

1. **Ask before writing or changing files.** Tell the user what you're about
   to create in one or two sentences and get a yes first.
2. **Never overwrite an existing file silently.** If one of the three files
   exists, copy it to `<name>.bak` first and say so.
3. **Write the files exactly as given below.** Don't "improve" them on the
   fly. If you believe something is wrong, tell the user what and why, and
   ask before deviating.
4. **Never ask for or write secrets** (API keys, tokens, passwords) into any
   file or into chat. This launcher stores none. Connections use
   environment variables or sign-in flows, as `Add-Connection-Guide.md` says.
5. **Report honestly.** Run the checks in Step 4 and show real results. Don't
   say it works until you've verified it.
6. These steps target Windows. If the OS is not Windows, stop and tell the
   user, because a `.bat` launcher won't run there. Offer a shell-script
   version instead and ask whether they want it.

### Step 1: Pre-flight

Run and summarise in plain English:

```powershell
$PSVersionTable.PSVersion
claude --version
codex --version
where.exe wt
Test-Path "$HOME\AI Sessions"
Get-ChildItem "$HOME\AI Sessions" -Directory | Select-Object -ExpandProperty Name
```

- If `claude` and `codex` are both missing, stop: the first setup guide has
  not been completed.
- If `$HOME\AI Sessions` is missing, ask to create it, plus `Work` and
  `Personal` inside it.
- `wt` (Windows Terminal) missing is fine. The launcher still works in a
  normal console window.

### Step 2: Write the files

Write these three files into `$HOME\AI Sessions`, preserving content exactly.

#### File 1: `AI-Launcher.bat`

```bat
@echo off
setlocal

rem Reopen inside Windows Terminal (nicer colors and fonts) if it is installed.
if not defined WT_SESSION (
  where wt >nul 2>nul
  if not errorlevel 1 (
    wt -d "%~dp0." --title "AI Launcher" cmd /k ""%~f0""
    exit /b
  )
)

powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0launcher.ps1" %*
if errorlevel 1 pause
```

#### File 2: `launcher.ps1`

```powershell
param([switch]$SelfTest)

# AI Launcher: pick an assistant and a folder, then chat or manage connections.
# Folders under this script's directory are the "areas" (Work, Personal, ...).
# Any .bat files in a "startup" folder next to this script run before chatting
# (use them to start local servers an MCP connection depends on).

$Root = $PSScriptRoot
$GuideRel = '..\Add-Connection-Guide.md'
$AddPrompt = 'Please read the file ..\Add-Connection-Guide.md and follow it to help me add a new MCP connection to this folder.'

function Test-Tool([string]$Name) {
  return $null -ne (Get-Command $Name -ErrorAction SilentlyContinue)
}

function Write-Banner {
  Clear-Host
  Write-Host ''
  Write-Host '  .--------------------------------.' -ForegroundColor Cyan
  Write-Host '  |      A I   T E R M I N A L     |' -ForegroundColor Cyan
  Write-Host '  ''--------------------------------''' -ForegroundColor Cyan
  Write-Host '   talk to your apps in plain English' -ForegroundColor DarkGray
  Write-Host ''
}

function Select-Option([string]$Title, [string[]]$Options) {
  Write-Host ''
  Write-Host " $Title" -ForegroundColor Green
  for ($i = 0; $i -lt $Options.Count; $i++) {
    Write-Host ("   [{0}] {1}" -f ($i + 1), $Options[$i]) -ForegroundColor White
  }
  while ($true) {
    $raw = Read-Host ' Pick a number'
    $n = 0
    if ([int]::TryParse($raw, [ref]$n) -and $n -ge 1 -and $n -le $Options.Count) {
      return ($n - 1)
    }
    Write-Host ' Please type one of the numbers above.' -ForegroundColor Yellow
  }
}

function Invoke-Startup {
  $startupDir = Join-Path $Root 'startup'
  if (-not (Test-Path $startupDir)) { return }
  Get-ChildItem -Path $startupDir -Filter '*.bat' | ForEach-Object {
    Write-Host " Running startup step: $($_.Name)" -ForegroundColor DarkGray
    & cmd.exe /c "`"$($_.FullName)`""
  }
}

# What is installed, and which folders exist?
$tools = @()
if (Test-Tool 'claude') { $tools += 'Claude' }
if (Test-Tool 'codex')  { $tools += 'Codex' }

$areas = @(Get-ChildItem -Path $Root -Directory |
  Where-Object { $_.Name -ne 'startup' } |
  Select-Object -ExpandProperty Name)

if ($SelfTest) {
  Write-Host "Root:        $Root"
  Write-Host ("Tools:       " + ($tools -join ', '))
  Write-Host ("Folders:     " + ($areas -join ', '))
  Write-Host ("Guide found: " + (Test-Path (Join-Path $Root 'Add-Connection-Guide.md')))
  Write-Host ("Startup dir: " + (Test-Path (Join-Path $Root 'startup')))
  if ($tools.Count -eq 0) { exit 1 }
  exit 0
}

if ($tools.Count -eq 0) {
  Write-Host ''
  Write-Host ' Neither Claude nor Codex was found on this computer.' -ForegroundColor Red
  Write-Host ' Run the first setup guide (AI-Terminal-Setup-Guide.md) first.' -ForegroundColor Yellow
  exit 1
}

if ($areas.Count -eq 0) {
  foreach ($a in 'Work', 'Personal') {
    New-Item -ItemType Directory -Path (Join-Path $Root $a) | Out-Null
  }
  $areas = @('Work', 'Personal')
}

while ($true) {
  Write-Banner

  if ($tools.Count -eq 1) {
    $tool = $tools[0]
    Write-Host " Assistant: $tool" -ForegroundColor DarkGray
  } else {
    $tool = $tools[(Select-Option 'Which assistant?' $tools)]
  }
  $cmd = $tool.ToLower()

  $area = $areas[(Select-Option 'Which folder?' $areas)]
  $areaPath = Join-Path $Root $area

  $action = Select-Option "What would you like to do in '$area'?" @(
    'Start chatting',
    'Add a new connection to an app',
    'See my connections',
    'Back'
  )

  switch ($action) {
    0 {
      Invoke-Startup
      Set-Location $areaPath
      Write-Host ''
      Write-Host " Starting $tool in $area ..." -ForegroundColor Green
      & $cmd
      exit 0
    }
    1 {
      Set-Location $areaPath
      Write-Host ''
      Write-Host " Starting $tool to help you add a connection ..." -ForegroundColor Green
      & $cmd $AddPrompt
      exit 0
    }
    2 {
      Push-Location $areaPath
      Write-Host ''
      Write-Host ' Checking connections (this can take a few seconds) ...' -ForegroundColor DarkGray
      & $cmd mcp list
      Pop-Location
      Write-Host ''
      Read-Host ' Press Enter to go back' | Out-Null
    }
    default { }
  }
}
```

#### File 3: `Add-Connection-Guide.md`

Write this file with the exact content between the `~~~` markers (not the
markers themselves):

~~~
# Add a connection (MCP server): instructions for the AI

You are helping a non-technical user connect ONE app to this folder. Your
working directory is the folder (for example Work or Personal). Go step by
step, plain language, one question at a time.

## Rules
- Ask before changing anything.
- NEVER ask the user to paste an API key, token or password into this chat.
  If a key is needed, give them the exact command to run in their own
  PowerShell, such as `setx MY_SERVICE_TOKEN "paste-key-here"`, and tell them
  to replace the placeholder. Never print or save a secret.
- Don't guess commands. If unsure, run `claude mcp add --help` (or
  `codex mcp --help`) or check the app's official docs first.
- Prefer MCP servers published by the app's own vendor. Local servers run
  code on this PC, so warn the user before installing anything obscure.
- If this is a work app, remind them once to check their employer's policy.

## Steps
1. Ask which app they want to connect.
2. Search the web or the app's developer docs for an official MCP server.
   Tell them what you found: hosted (a URL) or local (an npx command), and
   what sign-in it needs. If there is only an API and no MCP, say so and
   offer to call the API directly when they ask (key stored as an
   environment variable), or to build a small MCP server later.
3. Ask where to save it: this folder only (default `local`), shared in this
   folder's `.mcp.json` (`project`, never put secrets there), or every
   folder (`user`).
4. Add it.
   - Hosted with sign-in:
     `claude mcp add --transport http <name> <url> --scope <scope>`
   - Hosted with an API key held in an environment variable:
     `claude mcp add --transport http <name> <url> --header "Authorization: Bearer $env:MY_SERVICE_TOKEN"`
   - Local:
     `claude mcp add <name> --env API_KEY=$env:MY_SERVICE_KEY -- npx -y <package>`
   - For Codex, run `codex mcp --help` first, then use `codex mcp add` for
     local servers, or add a `[mcp_servers.<name>]` block with `url` (and
     `bearer_token_env_var = "<ENV_VAR_NAME>"` for key auth) to
     `~/.codex/config.toml`. Show the user the exact change before writing,
     and keep the rest of that file intact.
5. If it needs browser sign-in, tell the user to type `/mcp`, choose the
   server, then **Authenticate**. You cannot do this step for them.
6. Tell the user to type `/exit`, run AI-Launcher again, choose this folder
   and "Start chatting" (new connections load in a new session). Then try
   a harmless read-only request such as "show my 3 most recent items in
   <app>" and confirm it works.
7. Before the first time a connection is used to send, post, delete or
   change anything, remind them to read the permission prompt carefully.
8. If the server needs a helper program running first (rare), put a `.bat`
   file that starts it into the `startup` folder next to AI-Launcher.bat;
   the launcher runs those before chatting.

When finished, ask whether they want to connect another app.
~~~

### Step 3: Desktop shortcut (ask first)

Offer a desktop shortcut called "AI Launcher":

```powershell
$desktop = [Environment]::GetFolderPath('Desktop')
$ws = New-Object -ComObject WScript.Shell
$s = $ws.CreateShortcut("$desktop\AI Launcher.lnk")
$s.TargetPath = "$HOME\AI Sessions\AI-Launcher.bat"
$s.WorkingDirectory = "$HOME\AI Sessions"
$s.Save()
```

### Step 4: Verify (do not skip)

1. Syntax-check the script:

   ```powershell
   $errs = $null
   [void][System.Management.Automation.Language.Parser]::ParseFile("$HOME\AI Sessions\launcher.ps1", [ref]$null, [ref]$errs)
   $errs
   ```
   It must print nothing. If it prints errors, fix them (tell the user what
   you changed) and re-run.
2. Self-test without opening the menu:

   ```powershell
   powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\AI Sessions\launcher.ps1" -SelfTest
   ```
   Confirm it lists the installed tool(s), the folders, and `Guide found: True`.
3. Do NOT launch the interactive menu yourself. It takes over the terminal.
   Ask the user to double-click **AI Launcher** (or `AI-Launcher.bat`) and
   confirm the menu appears and "See my connections" runs.

### Step 5: Wrap up

Tell the user, in short plain language:
- What was created and where.
- How to use it: double-click AI Launcher, pick assistant and folder, then
  "Add a new connection to an app" whenever they want to connect something.
- That the menu's "See my connections" option is a quick status check.
- Anything that failed or still needs their attention.

### Troubleshooting reference

| Problem | What to do |
|---|---|
| Window flashes and closes | Run `AI-Launcher.bat` from a PowerShell window to see the error. |
| "running scripts is disabled" | The `.bat` already passes `-ExecutionPolicy Bypass`. If you ran `launcher.ps1` directly, use the `.bat` instead. |
| Menu shows only one assistant | The other CLI isn't installed or isn't on PATH. Reopen the terminal, or install it (see the first guide). |
| Added connection doesn't show up | It loads in a new session. Exit and relaunch from the same folder. |
