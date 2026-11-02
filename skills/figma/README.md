# Figma Skill for GitHub Copilot

Turn Figma designs into code directly from Copilot chat, with no MCP server. Paste a Figma link and the agent pulls the frame's layout, colors, typography, and a rendered reference image, then implements it with your repo's existing components.

Uses a Figma personal access token (read-only) and calls the Figma REST API directly.

## What it can do

- Fetch a frame's node tree: auto-layout, spacing, padding, sizing
- Extract colors, strokes, corner radii, shadows, and text styles
- Render a frame as PNG for visual reference
- Export icons and assets as SVG
- List a file's pages, frames, published styles, and components
- Read file comments

## Writing to Figma

The token and REST API are **read-only**, so reads and writes work differently. Writing uses the Figma Plugin API with no MCP server. It needs a **Full seat** with edit access to a Design file; a Dev seat cannot write. Pick one method:

| | Option A: Browser | Option B: Figma Desktop bridge |
|---|---|---|
| Install | Nothing | Dev plugin + Node |
| Plugin to keep open | No | Yes |
| Files at once | One per browser tab | One |
| Sign-in | The browser's Figma session | Figma Desktop |
| Relies on | The `figma` global in the web editor page (undocumented) | Documented plugin APIs |

### Option A: Browser (no install)

The agent opens the file at its `figma.com/design/...` URL in a browser it can control, then runs `figma.*` JavaScript in the page and checks the result with a screenshot. Sign in to Figma yourself in that browser, never in chat. The method is the same in every client; only the browser tool differs:

| Client | Browser tool |
|---|---|
| GitHub Copilot app | Browser canvas (`open_canvas`, then `evaluate_javascript`) |
| Claude Code / Claude desktop app | Claude in Chrome (your Chrome, already signed in) or the built-in browser (`javascript_tool`) |
| Copilot CLI, VS Code chat | No browser tool; use Option B |

Each file can be open in its own tab, so one agent, or several agent sessions, can work on different files at once.

### Option B: Figma Desktop bridge

A dev plugin in Figma Desktop talks to a small local server (both in `bridge/`), so the agent can write directly and read results back.

One-time setup:

1. In Figma Desktop: **Plugins > Development > Import plugin from manifest...** and choose `bridge/manifest.json` from the installed skill folder. The Plugins menu only appears inside an open Design file.

Each session:

1. Open a Design file you can edit and run **Plugins > Development > Copilot Figma Bridge**. Keep it open.
2. In your Copilot client, ask the agent to start the bridge (`node bridge/server.js`). It prints a token.
3. Paste the token into the plugin window and click **Connect**. The plugin saves it and reconnects with it next time, until the server restarts. The label can take up to 25 seconds to change to "Connected".
4. Ask Copilot to make the change, for example "create a card frame in this file". The agent sends Plugin API scripts to the local server and the plugin runs them.
5. Stop the server when you are done.

Figma closes the plugin when you close its window, switch files, open Dev Mode, or run another plugin or AI action. Reopen it with **Ctrl+Alt+P** (Run last plugin; **Cmd+Option+P** on Mac).

Requires Node. The server listens on `127.0.0.1` only and needs the token, but it runs code in your open file, so use it on files you trust.

### What the Plugin API can't do

File-level actions such as renaming, moving, or deleting a file are not in the Plugin API or the REST API. Use the file menu (the ⌄ next to the file name) instead.

Details for both options are in "Writing to Figma" in [SKILL.md](./SKILL.md). For Manulife CDS, pair with the `mux-figma` skill from `manulife-innersource/mux-core`.

## How writing works

Figma web and Figma Desktop run the same editor, loaded from figma.com. Each open file gets its own editor, made of:

- a **document engine** (C++ compiled to WebAssembly) that holds the file in memory and draws the canvas,
- the **Plugin API**, the `figma` object: a JavaScript interface to that engine (`figma.createFrame()`, `node.characters = ...`),
- a **multiplayer client** that sends every change over a WebSocket to Figma's servers.

Every edit, from the mouse, a menu, or a script, becomes a change in the engine and is synced and saved by the server. There is no Save button.

**Reaching the Plugin API.** The API is the same everywhere; what differs is how outside code gets to it:

```
Option A, browser:
  agent -> browser tool -> JavaScript in the tab -> window.figma -> engine

Option B, Desktop bridge:
  agent -> shell -> server.js (localhost:9230) <- polls - plugin UI (iframe)
                                                             | postMessage
                                                   plugin sandbox -> figma -> engine
```

- **Browser.** The web editor also exposes the Plugin API on the page as `window.figma`, a convenience for plugin developers testing in the console. Browsers let outside tools run JavaScript in a page, through extensions or automation over the DevTools Protocol, so the agent can call it directly. This is undocumented and could change, and there is no sandbox, so a script runs with your full access to the file.
- **Desktop.** Figma Desktop is a dedicated browser shell (Electron) with no extensions and no automation port, so a plugin is the only supported way in. A normal plugin runs its logic in a sandbox that only sees `figma`, and its UI in an iframe that can use the network, limited to the domains in `manifest.json`. The bridge uses that iframe to poll the local server for scripts. This is why the plugin must stay open, and why it serves only the file it runs in.

**Multiple sessions.** Each open file has its own room on Figma's servers. Every editor with the file open (browser tabs, Desktop, teammates) keeps a local copy and a live connection to that room. Changes are sent as small property updates and broadcast to everyone in the room.

- **Different files:** each tab has its own engine and its own connection, so agent sessions working in separate files never interact. All tabs share the browser's cookies, so one sign-in covers them. This is the recommended setup.
- **Same file:** this works through multiplayer, but if two sessions change the same property of the same layer, the last write wins. Give each session its own file, or at least its own page.
- **Limits are local:** every tab is a full editor, so open a few, not dozens. Plugin API calls run in the local editor and are not rate-limited like the REST API (`429`). Browsers slow down hidden tabs, so run heavy scripts in the visible tab.
- **The Desktop bridge is single-file:** it has one server on one port, and hands each script to whichever plugin polls first.


## Why not the Figma MCP server?

Figma's official MCP server (remote `https://mcp.figma.com/mcp` or the desktop Dev Mode server) is the richer option where it's allowed. Enterprise Copilot policies (like Manulife's) often block third-party MCP servers, so the Figma tools never appear. This skill needs only outbound HTTPS to `api.figma.com`.

## Before you install: set up your Figma token

1. **Create the token.** In Figma, click your avatar > **Settings > Security > Personal access tokens > Generate new token**. Set scope **File content: Read-only** (everything else **No access**), choose an expiry, and copy the token. It is shown only once.
2. **Store it on your computer** as `FIGMA_TOKEN`. Don't paste it into chat.
   - Windows (PowerShell): `[Environment]::SetEnvironmentVariable('FIGMA_TOKEN','<token>','User')`
   - Mac: `echo 'export FIGMA_TOKEN=<token>' >> ~/.zshrc`
   - Linux: `echo 'export FIGMA_TOKEN=<token>' >> ~/.bashrc`
3. **Restart your Copilot client** so it sees the new variable.

Full steps (including a Windows GUI method, verification, and rotation) are in [INSTALL.md](./INSTALL.md#step-1-create-and-store-a-figma-token).

## Install

Pick your client and paste the matching prompt into your Copilot chat (Agent mode):

### VS Code Copilot Chat

```
Follow the install instructions at
https://github.com/stevez-manulife/copilot-skills/blob/main/skills/figma/vscode/install.prompt.md
```

### GitHub Copilot CLI

```
Follow the install instructions at
https://github.com/stevez-manulife/copilot-skills/blob/main/skills/figma/copilot-cli/install.prompt.md
```

### GitHub Copilot Desktop App

```
Follow the install instructions at
https://github.com/stevez-manulife/copilot-skills/blob/main/skills/figma/copilot-app/install.prompt.md
```

### Claude Code

```
Follow the install instructions at
https://github.com/stevez-manulife/copilot-skills/blob/main/skills/figma/claude-code/install.prompt.md
```

The agent installs the skill to the correct folder for your client and verifies your Figma token.

> Requires **Agent mode** (terminal access).

For manual install steps, see [INSTALL.md](./INSTALL.md).

## Usage

Copy a frame link in Figma (right-click > **Copy link to selection**), then ask:

```
implement this design as a React component: https://www.figma.com/design/<key>/<name>?node-id=12-345
what colors and fonts does this frame use? <figma link>
export the icons in this frame as SVG: <figma link>
```

## Requirements

- A Figma account with access to the file
- A Figma personal access token with **File content: Read-only** scope, stored in the `FIGMA_TOKEN` environment variable (never paste it into chat)
- `jq` for the Mac/Linux examples (optional; PowerShell parses JSON natively)
