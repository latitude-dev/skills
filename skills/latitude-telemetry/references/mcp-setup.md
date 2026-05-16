# Latitude MCP server: setup and use during onboarding

Latitude ships a remote, OAuth-authenticated MCP server at `https://api.latitude.so/v1/mcp`. Once connected, your coding agent (Claude Code, Cursor, Codex, …) can call Latitude API methods directly from inside the conversation — list projects, create projects, list traces, create annotations, manage API keys, and so on. The tool catalog is generated from the Latitude API surface, so it stays in sync with the platform.

Source of truth for install steps: [docs.latitude.so/getting-started/mcp](https://docs.latitude.so/getting-started/mcp). This file is a working-skill summary; if the docs have drifted, the docs win.

## Why this matters during the telemetry install

This skill frequently needs one or more Latitude projects to exist **before** traces can flow. The common cases:

- The user has a brand-new account with no projects yet.
- The user has confirmed they want [multi-project routing](project-scoping.md) and one or more of the secondary slugs do not exist.
- The user wants to verify that a slug they think exists actually resolves before any SDK code is written.

Without the MCP installed, all of these require the user to leave the conversation, open the console, click around, and paste a slug back. With the MCP installed, the agent can call `listProjects` / `createProject` directly and read the slug straight back. The skill works either way — the MCP just removes a context switch.

**The MCP is for the agent, not for the running app.** Once your service is up, it still uses `LATITUDE_API_KEY` to send traces over OTLP. The MCP only authenticates *you, the agent*, so you can perform setup actions on the user's behalf during this conversation. Don't conflate the two — installing the MCP is not a substitute for setting `LATITUDE_API_KEY` in the app's environment.

## Step 1 — Detect the client *before* suggesting a command

The install command depends on which assistant the user is running you in. Use these signals; do not guess.

| Client | Signals you can use |
| --- | --- |
| **Claude Code CLI** | The CLI's own system context identifies it as "Claude Code" (this is the CLI itself); config lives in `~/.claude.json`. |
| **Claude Desktop app** | macOS / Windows desktop client; config at `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows). |
| **Cursor** | Cursor IDE; config in `~/.cursor/mcp.json`. |
| **Codex (CLI / desktop)** | OpenAI Codex; config in `~/.codex/config.toml`. |
| **Gemini CLI** | Google Gemini CLI; config in `~/.gemini/settings.json`. |
| **Zed** | Zed editor with AI features; config in `~/.config/zed/settings.json`. |
| **OpenCode** | `opencode` CLI binary is on `PATH`. |
| **Google Antigravity** | Config in `~/.antigravity/mcp.json`. |
| **GitHub Copilot** | Surface varies (VS Code Chat, Copilot Coding Agent, Copilot CLI). Check Copilot's per-surface MCP docs. |
| **Other / unknown** | Ask the user once: *"Which assistant are you running me in?"* Then use the matching block below. |

If the system prompt or environment already tells you which client you are (it usually does), trust it. If it's genuinely ambiguous, ask once before suggesting any command — pasting a Claude-Code-only command into a Cursor config does nothing useful.

## Step 2 — Install the MCP server (per-client commands)

The server URL is the same for every client: `https://api.latitude.so/v1/mcp`. Auth is OAuth — the first MCP call surfaces a browser prompt the user must complete; mention this so they don't think it's hanging.

### Claude Code CLI

```bash
claude mcp add --transport http latitude https://api.latitude.so/v1/mcp --scope user
```

Verify with `claude mcp list`. The first call to a Latitude MCP tool triggers the OAuth flow in the browser.

Equivalent manual edit of `~/.claude.json`:

```json
{
  "mcpServers": {
    "latitude": {
      "type": "http",
      "url": "https://api.latitude.so/v1/mcp"
    }
  }
}
```

### Claude Desktop

Claude Desktop doesn't yet speak remote MCP natively; bridge via the `mcp-remote` shim. Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "latitude": {
      "command": "npx",
      "args": ["mcp-remote", "https://api.latitude.so/v1/mcp"]
    }
  }
}
```

The user must **restart Claude Desktop** for the config change to load.

### Cursor (desktop & CLI)

`~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "latitude": {
      "url": "https://api.latitude.so/v1/mcp"
    }
  }
}
```

For Cursor's CLI agent: `agent mcp login latitude` to complete OAuth.

### Codex (desktop & CLI)

`~/.codex/config.toml`:

```toml
[mcp_servers.latitude]
transport = "http"
url = "https://api.latitude.so/v1/mcp"
```

### Gemini CLI

`~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "latitude": {
      "httpUrl": "https://api.latitude.so/v1/mcp"
    }
  }
}
```

### Zed

`~/.config/zed/settings.json`:

```json
{
  "context_servers": {
    "latitude": {
      "url": "https://api.latitude.so/v1/mcp"
    }
  }
}
```

Restart Zed after editing.

### OpenCode

```bash
opencode mcp add
# Answer:
#   name = latitude
#   type = remote
#   url  = https://api.latitude.so/v1/mcp
#   oauth = yes

opencode mcp auth latitude
```

### Google Antigravity

`~/.antigravity/mcp.json`:

```json
{
  "mcpServers": {
    "latitude": {
      "url": "https://api.latitude.so/v1/mcp"
    }
  }
}
```

### GitHub Copilot

Copilot's MCP support is split across Copilot Chat in VS Code, Copilot Coding Agent, and Copilot CLI, and each has its own settings surface that changes faster than this file does. Point the user at [docs.latitude.so/getting-started/mcp](https://docs.latitude.so/getting-started/mcp) and have them follow the Copilot-specific section there. Do not invent a config path.

### Anything else / unknown

Fall back to the docs page above and let the user follow the per-client instructions. Don't paste a config that wasn't validated for their client.

## Step 3 — Verify connection

After install (and any required restart), confirm the agent has access to Latitude MCP tools. The exact tool-name prefix varies per client (in Claude Code they look like `mcp__latitude__listProjects` etc.); the *capabilities* are the same.

Quick smoke test: call `listProjects`. If it returns a list (even empty), you're connected. If it errors with "not authenticated" or similar, the OAuth flow didn't complete — have the user re-trigger it in the browser.

## Step 4 — Use the MCP for end-to-end setup

With the MCP connected, the agent can drive **both** halves of the credentials setup (API key and project slug) without the user leaving the editor. This is the fully automated path. The user can still opt to do anything manually — don't force automation if they decline.

Useful tools for the telemetry install flow:

| Need | Tool | Returns the secret? |
| --- | --- | --- |
| See what projects exist in this org | `listProjects` | n/a — projects have no secret |
| Create a new project | `createProject` (takes a name; slug is generated from it) | n/a |
| Confirm a specific slug resolves | `getProject` | n/a |
| See what API keys already exist in this org | `listApiKeys` | **no** — only metadata (name, prefix, last 4 chars, created-at) |
| Create a new API key | `createApiKey` (takes a name) | **yes — once, in the create response.** Cannot be retrieved again later. |
| Inspect an existing key's metadata | `getApiKey` | **no** — metadata only |
| Revoke a key | `revokeApiKey` | n/a |
| Confirm which org the OAuth session is bound to | `getAccount` or `listApiKeys` | n/a |

### Recommended end-to-end flow

Steps 1–2 handle the API key. Steps 3–4 handle projects. Run them in order so you have the API key before verifying that slugs resolve.

1. **Inventory existing API keys with `listApiKeys`.** Show the list to the user (names, prefixes, last 4 chars) so they can decide:
   - *"Use an existing key — I'll paste the secret"* → ask the user to paste it (you have no way to retrieve the secret of an old key). They paste into the chat; you walk them through writing it to `.env` per Rule 5.
   - *"Create a new one for this install"* → continue to step 2.
   - *No keys exist yet* → automatically suggest creating one, name suggestion `telemetry-<service-name>` or similar, but **confirm the name with the user before calling `createApiKey`**.
2. **Call `createApiKey` with the confirmed name.** Capture the full secret from the response — it will not be retrievable again. Then **ask the user which they prefer**:
   - *"Write the new key to `.env` for me"* → with explicit confirmation in the chat, you may write it. See the Rule 5 carve-out in `SKILL.md`. Echo the file path you wrote to. Don't write any other secrets the user didn't explicitly authorize.
   - *"Show me the value and I'll paste it myself"* → surface the value in the chat, give them the exact line to add, and have them paste. This is the safer default for users who prefer to handle their own secrets.
3. **Inventory existing projects with `listProjects`.** For each slug the install needs:
   - If it already exists → record it.
   - If it doesn't → confirm the project **name** with the user (don't auto-name), then call `createProject`. Read the slug back from the response.
4. **Write or surface the slugs** the same way as the API key, per Rule 5 and its MCP carve-out. Slugs are not secrets but the same write-confirmation pattern applies — ask the user before modifying `.env`.

At the end of this sequence, the user has a real `LATITUDE_API_KEY` and one or more real `LATITUDE_PROJECT_SLUG` values, either written to `.env` (with their consent) or sitting in the chat ready to paste. Continue with the curl probe in `SKILL.md` Step 1d to confirm the credentials reach Latitude before writing SDK code.

### Don't skip listing first

Always call `listApiKeys` and `listProjects` before creating anything. Two reasons:

1. **Avoid duplicates.** Users frequently already have a `local-dev` or `telemetry` key from a previous attempt; creating a second one clutters the keys page and wastes a slot.
2. **Catch wrong-org OAuth early.** If the lists are surprisingly empty — or contain names the user doesn't recognize — the OAuth session bound to the wrong org during install. Surface this before creating anything new in the wrong workspace.

## Step 5 — Inform the user about ongoing value

The MCP is useful past the install itself: once connected, the user (via you, the agent) can list traces, fetch specific spans, create annotations, manage saved searches, invite team members, rotate API keys — all without leaving the editor. Mention this briefly after install so they know the install wasn't a one-shot setup detour.

## When the user declines the MCP

Don't push. The manual path works just as well, just with more clicks:

1. Send them to [https://console.latitude.so](https://console.latitude.so) → **New project** for each project they need.
2. Have them paste the resulting slug back into the chat.
3. Continue.

If they're worried about the OAuth scope (the MCP's OAuth key authorizes the agent to perform any Latitude API action the user can — including destructive ones like `revokeApiKey` or `deleteProject`), tell them: the OAuth key is listed under **Settings → Keys → OAuth Keys** in the console and can be revoked at any time, including the moment this skill finishes.

## Pitfalls

- **Wrong-org OAuth.** Users with multiple Latitude orgs can sign into the wrong one during the OAuth flow, in which case `listProjects` shows the wrong list. If a slug "exists" but the MCP can't see it, check **Settings → Keys → OAuth Keys** for which org the connected agent is bound to.
- **Forgot to restart.** Claude Desktop, Cursor desktop, and Zed all require a restart after editing their config files. The first MCP call after install will look like it's hanging when really the host hasn't loaded the new config.
- **Pretending MCP install is mandatory.** It isn't. If the user prefers the console, that's fine — the install completes either way. Don't gate the telemetry install on MCP install.
- **Conflating MCP auth with app auth.** Installing the MCP does not set `LATITUDE_API_KEY` in the user's app. Those are different credentials with different scopes. The app still needs its own API key in `.env` (or wherever it loads env vars from).
- **Auto-picking project names.** `createProject` takes a name and derives a slug. Names are user-facing — confirm them in the conversation before creating. Avoid auto-generating like `agent-1`, `service-foo`; the user has to live with the name later.
