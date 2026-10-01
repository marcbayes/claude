# AI Terminal Setup: instructions for Claude Code

> **For the human:** this file is meant to be read by Claude Code, not by you.
> Open a Claude Code session (see "Before you start" below), drop this file
> into it, and say: **"Follow the instructions in this file."** Claude will
> do the setup with you, step by step, and ask before anything important.
>
> **Before you start (one time, about 5 minutes):** Claude Code has to exist
> before it can read this file. Install Node.js from https://nodejs.org (LTS),
> then in PowerShell run `npm install -g @anthropic-ai/claude-code`, then run
> `claude` and log in with your own Claude account. Then drop this file in.

---

## Instructions for Claude (the agent)

You are setting up a new user's computer so they can use AI in a terminal
with connections ("MCP servers") to the apps they use. The user is not
necessarily technical. Be friendly, plain-spoken, and go one step at a time.

### Ground rules

1. **Ask before installing or changing anything.** Say what you're about to
   do in one sentence, then do it once they agree. A single "yes, go ahead
   with the installs in Phase 2" is enough for that whole phase.
2. **Never ask the user to paste an API key, token or password into this
   chat.** Secrets belong in environment variables or in the app's own
   sign-in flow. If a key is needed, give the user the exact command to run
   themselves in their own PowerShell window (for example
   `setx MY_SERVICE_TOKEN "paste-key-here"`) and tell them to replace the
   placeholder. Never print, log or write a secret into a file you create.
3. **Don't guess commands.** If you're unsure of a current flag or config
   format (especially for Codex), run `<tool> --help` or `<tool> mcp --help`
   and read it, or check the official docs, before acting.
4. **Check the OS first.** These steps target Windows (PowerShell). If the
   machine is macOS or Linux, adapt the commands (Homebrew or the system
   package manager instead of `winget`, shell profile instead of `setx`) and
   tell the user you're doing so.
5. **Work-account caution.** If the user mentions connecting work apps,
   remind them once to check their employer's policy on AI tools before
   connecting company data.
6. **Third-party local servers run code on this PC.** Only install MCP
   servers published by the app's own vendor or well-known maintainers. If
   the user asks for something obscure, say so and let them decide.
7. **Report honestly.** If a step fails, show the actual error summary and
   what you'll try. Don't claim something works until you've verified it.
8. **Don't read, print or modify files outside the setup folders and the
   MCP/CLI config files you're explicitly configuring.**

### Phase 1: Check what's already there

Run these and summarise the results in plain English:

```powershell
$PSVersionTable.PSVersion
node --version
npm --version
git --version
claude --version
codex --version
winget --version
```

Missing commands are expected; just note which are missing. Then ask:

- "Do you want Claude Code only, or Codex (OpenAI) as well?" (Claude Code is
  already running, so it's installed. Codex is optional and needs a ChatGPT
  or OpenAI account.)
- "Which apps do you want to connect? Give me a rough list, work and
  personal. Not sure is fine; we can add them later."

### Phase 2: Install what's missing

After the user agrees, install only what's missing:

```powershell
winget install OpenJS.NodeJS.LTS      # if node/npm missing
winget install Git.Git                # if git missing
winget install Microsoft.WindowsTerminal   # optional, nicer terminal
npm install -g @openai/codex          # only if the user wants Codex
```

Notes:
- After installing Node or Git, the current shell may not see the new
  commands. If so, tell the user to close and reopen the terminal, restart
  `claude`, and say "continue setup." Resume from this phase.
- If `winget` itself is missing, point the user to the Microsoft Store "App
  Installer" or https://nodejs.org and https://git-scm.com and wait.
- If Codex was installed, tell the user to run `codex` once in their own
  terminal to log in (it opens a browser), then come back.

### Phase 3: Create the home folders

Create (skip any that exist):

```powershell
mkdir "$HOME\AI Sessions"
mkdir "$HOME\AI Sessions\Work"
mkdir "$HOME\AI Sessions\Personal"
```

Explain: connections can be saved per folder, so work tools stay in `Work`
and personal tools stay in `Personal`. Skip the split if the user prefers one
folder.

### Phase 4: Connect their apps (repeat per app)

For each app on the user's list:

1. **Find out if it has an MCP server.** Search the web / the app's developer
   docs for an official MCP server and its URL or install command. Tell the
   user what you found and whether it is hosted (a URL) or local (an `npx`
   command). If there is no MCP, only an API, jump to "API-only apps" below.
2. **Decide the scope** with the user: `local` (this folder only, default),
   `project` (saved in the folder's `.mcp.json`, shareable, so never put
   secrets in it), or `user` (every folder).
3. **Add it** from the right folder, using Claude Code's built-in command:

   Hosted, sign-in based:
   ```powershell
   claude mcp add --transport http <name> <url> --scope <scope>
   ```
   Hosted, API-key based (the key comes from an environment variable the user
   set themselves, never typed into chat):
   ```powershell
   claude mcp add --transport http <name> <url> --header "Authorization: Bearer $env:MY_SERVICE_TOKEN"
   ```
   Local server:
   ```powershell
   claude mcp add <name> --env API_KEY=$env:MY_SERVICE_KEY -- npx -y <package>
   ```
   (Run `claude mcp add --help` first if any flag looks different from this.)
4. **Sign-in:** if the server needs browser sign-in, tell the user to type
   `/mcp` in a Claude Code session, select the server and choose
   **Authenticate**. You can't do this step for them.
5. **Verify:** run `claude mcp list` and report the status. Newly added
   servers usually load only in a *new* session, so tell the user to restart
   `claude` in that folder, then run a harmless read-only test (for example
   "list my 3 most recent items from <app>") and report whether it worked.
6. **If Codex is also installed**, offer to add the same connection there.
   Run `codex mcp --help`, then use `codex mcp add` for local servers, or add
   a `[mcp_servers.<name>]` block with `url` and, for key-based auth,
   `bearer_token_env_var = "<ENV_VAR_NAME>"` to `~/.codex/config.toml`. Show
   the user the exact change before writing it. Keep existing content in that
   file intact.

Start with read-only tests. Before the first time any connection is used to
send, post, delete or change something, remind the user to read the
permission prompt carefully.

### API-only apps

If an app has no MCP server, tell the user, then offer to either (a) call its
API directly when asked, using the key from an environment variable and the
app's API docs, or (b) later build a small MCP server for it. Don't build (b)
during setup unless the user asks.

### Phase 5: Desktop shortcuts (optional)

Offer to create shortcuts that open Windows Terminal in the right folder.
Only do this if the user wants them:

```powershell
$desktop = [Environment]::GetFolderPath('Desktop')
$ws = New-Object -ComObject WScript.Shell
foreach ($area in 'Work','Personal') {
  $s = $ws.CreateShortcut("$desktop\Claude $area.lnk")
  $s.TargetPath = "wt.exe"
  $s.Arguments = "-d `"$HOME\AI Sessions\$area`" cmd /k claude"
  $s.Save()
}
```

(For Codex shortcuts, repeat with `codex` in place of `claude`.) If `wt.exe`
isn't installed, use `cmd.exe /k "cd /d ... && claude"` instead.

### Phase 6: Wrap up

Give the user a short summary:

- What was installed and what was already there.
- Which connections are working, and which still need a sign-in or a restart.
- How to start each day: open the shortcut (or open a terminal, `cd` into
  `Work` or `Personal`, type `claude`), then just talk normally.
- Handy commands: `/mcp` (connection status and sign-in), `claude mcp list`,
  `claude mcp remove <name>`, `/help`.
- Any follow-ups you couldn't finish, as a plain list.

### Troubleshooting reference

| Problem | What to do |
|---|---|
| `claude` or `codex` not recognized | Close and reopen the terminal. Still failing: run `npm config get prefix` and check that folder is on PATH. |
| Connection failed in `/mcp` | Re-authenticate there. For local servers, run the `npx` command by hand to see the error. |
| Tools missing in a session | Wrong folder for the scope used, or the session started before the server was added. Restart in the right folder, or re-add with `--scope user`. |
| Start over on one connection | `claude mcp remove <name>`, then add again. |
