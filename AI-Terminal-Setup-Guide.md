# AI Terminal Setup Guide (Windows)

Talk to Claude or Codex in a terminal exactly like a normal chat, but with the
power to use your real apps (calendar, drive, CRM, design tools, etc.) through
**MCP connections**.

Time needed: about 20-30 minutes. Nothing here needs coding.

---

## The 30-second explanation

- **The CLI** (Claude Code or Codex) is the chat window in your terminal. You
  type normally. It decides when to use tools.
- **An MCP server** is a plug that gives the AI access to one app. Add it once
  and every future session can use it.
- **An API** is the app's raw interface. MCP servers are usually wrappers
  around an API. If an app has only an API, see Part 6.
- You never "pick" which MCP to use. Connect several, then just ask
  ("check my calendar, then find last month's invoices"). The AI chooses.

---

## Part 1: Install the basics (one time)

Open **PowerShell** (Start menu, type "PowerShell") and run:

```powershell
winget install OpenJS.NodeJS.LTS
winget install Git.Git
winget install Microsoft.WindowsTerminal
```

Close and reopen PowerShell, then check:

```powershell
node --version
git --version
```

Both should print a version number.

## Part 2: Install the AI tool(s)

You can install one or both.

```powershell
npm install -g @anthropic-ai/claude-code
npm install -g @openai/codex
```

Log in (each opens a browser; use your own account):

```powershell
claude        # follow the login prompt, then type /exit
codex         # follow the login prompt
```

Requires your own Claude plan (Claude Code) and/or your own OpenAI/ChatGPT
account (Codex). Logins are personal and can't be shared.

## Part 3: Make a home folder

```powershell
mkdir "$HOME\AI Sessions"
mkdir "$HOME\AI Sessions\Work"
mkdir "$HOME\AI Sessions\Personal"
```

Why separate folders? Connections can be saved per folder, so Work tools stay
in `Work` and Personal tools stay in `Personal`. (See scopes below.)

## Part 4: Add a connection (MCP)

First, find out whether the app has an MCP server. Easiest way: start a
session and ask, "Does <app> have an MCP server, and how do I connect it?"
Or check the app's developer docs / the Claude connector directory.

### Claude Code

Start in the right folder:

```powershell
cd "$HOME\AI Sessions\Work"
```

**Hosted (remote) MCP**, the most common and easiest. You just need a URL:

```powershell
claude mcp add --transport http <name> <url>
```

Example shape: `claude mcp add --transport http notion https://example.com/mcp`

Then start `claude` and type `/mcp`. Pick the server and choose
**Authenticate** if it needs a sign-in (a browser opens).

**Hosted MCP that uses an API key instead of sign-in:**

```powershell
claude mcp add --transport http <name> <url> --header "Authorization: Bearer YOUR_KEY"
```

**Local MCP** (runs a program on your PC, usually via npx):

```powershell
claude mcp add <name> --env API_KEY=YOUR_KEY -- npx -y <package-name>
```

**Manage connections:**

```powershell
claude mcp list
claude mcp remove <name>
```

**Scopes** (where the connection is saved). Add `--scope` to the add command:

| Scope | Meaning |
|---|---|
| `local` (default) | Only you, only this folder |
| `project` | Saved in the folder in a `.mcp.json` file (shareable) |
| `user` | You, in every folder |

Tip: put work tools in the `Work` folder, personal tools in `Personal`.
Never put API keys in a `project` scope file you'll share or upload anywhere.

### Codex

Codex keeps its list in `%USERPROFILE%\.codex\config.toml`.

```powershell
codex mcp add <name> -- npx -y <package-name>      # local server
codex mcp list
codex mcp --help                                    # confirm current options
```

For a hosted server, edit `config.toml` and add:

```toml
[mcp_servers.<name>]
url = "https://example.com/mcp"
bearer_token_env_var = "MY_SERVICE_TOKEN"
```

Then store the token as a user environment variable (not in the file):

```powershell
setx MY_SERVICE_TOKEN "YOUR_KEY"
```

Close and reopen the terminal afterward. Codex's MCP options change often, so
if something here doesn't match, run `codex mcp --help` and check the Codex docs.

## Part 5: Daily use

1. Open Windows Terminal.
2. `cd "$HOME\AI Sessions\Work"` (or `Personal`).
3. Type `claude` (or `codex`).
4. Talk normally.

Useful in Claude Code: `/mcp` shows connections and their status. `/help`
lists everything else.

**Optional: a desktop shortcut.** Right-click Desktop, New, Shortcut, and set
the location to:

```
wt -d "%USERPROFILE%\AI Sessions\Work" cmd /k claude
```

Name it "Claude Work". Make another for Personal, and for Codex swap `claude`
for `codex`. That is all the old launcher files really did.

## Part 6: When an app has no MCP, only an API

- Ask the AI to call it. Give it the API docs link and the key (store the key
  as an environment variable, e.g. `setx MY_APP_KEY "..."`), then say "use the
  API to pull X." The CLI can run the requests itself.
- For something you'll use constantly, ask it to build a small MCP server for
  that app. Do this later, once the basics work.

---

## Safety checklist

- [ ] **Work data:** check company policy before connecting work apps to any AI.
- [ ] **Third-party local MCP servers run code on your PC.** Only install ones
      from sources you trust (the app's own vendor, or well-known projects).
- [ ] **Keys:** never paste API keys into chats, emails, or shared files. Use
      environment variables.
- [ ] **Permissions:** the CLI asks before it runs commands or changes things.
      Read the prompt before approving, especially for tools that send, delete,
      or post.
- [ ] Start with read-only tasks ("show me", "summarize") until you trust a
      connection.

## Troubleshooting

| Problem | Fix |
|---|---|
| `claude` / `codex` not recognized | Close and reopen the terminal. If still failing: `npm config get prefix`, then make sure that folder is on your PATH. |
| Connection shows failed in `/mcp` | Re-authenticate there. For local servers, run the `npx` command by hand to see the error. |
| Tools don't appear | You may be in a different folder than where you added the connection (scope). Use `--scope user` for "everywhere." |
| Need to start over on one connection | `claude mcp remove <name>`, then add again. |
