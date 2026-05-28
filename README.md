# OpenCode plugin for Obsidian


Give your notes AI capability by embedding Opencode [OpenCode](https://opencode.ai) AI assistant directly in Obsidian:

<img src="./assets/opencode_in_obsidian.png" alt="OpenCode embeded in Obsidian" />

**Use cases:**
- Summarize and distill long-form content
- Draft, edit, and refine your writing
- Query and explore your knowledge base
- Generate outlines and structured notes

This plugin uses OpenCode's web view that can be embedded directly into Obsidian window. Usually similar plugins would use the ACP protocol, but I want to see how how much is possible without having to implement (and manage) a custom chat UI - I want the full power of OpenCode in my Obsidian.

_Note: plugin author is not afiliated with OpenCode or Obsidian - this is a 3rd party software._

## Requirements

- Desktop only (uses Node.js child processes)
- [OpenCode CLI](https://opencode.ai) installed 
- [Bun](https://bun.sh) installed

## Installation

### For Users (BRAT - Recommended for Beta Testing)

The easiest way to install this plugin during beta is via [BRAT](https://github.com/TfTHacker/obsidian42-brat) (Beta Reviewer's Auto-update Tool):

1. Install the BRAT plugin from Obsidian Community Plugins
2. Open BRAT settings and click "Add Beta plugin"
3. Enter: `mtymek/opencode-obsidian`
4. Click "Add Plugin" - BRAT will install the latest release automatically
5. Enable the OpenCode plugin in Obsidian Settings > Community Plugins

BRAT will automatically check for updates and notify you when new versions are available.

### For Developers

If you want to contribute or develop the plugin:

1. Clone to `.obsidian/plugins/obsidian-opencode` subdirectory under your vault's root:
   ```bash
   git clone https://github.com/mtymek/opencode-obsidian.git .obsidian/plugins/obsidian-opencode
   ```
2. Install dependencies and build:
   ```bash
   bun install && bun run build
   ```
3. Enable in Obsidian Settings > Community Plugins
4. Add AGENTS.md to your workspace root to guide the AI assistant

## Usage

- Click the terminal icon in the ribbon, or
- `Cmd/Ctrl+Shift+O` to toggle the panel
- Server starts automatically when you open the panel


## Settings

### Custom Command Mode

Enable "Use custom command" when you need more control over how OpenCode starts—for example, to add extra CLI flags, use a custom wrapper script, or run OpenCode through a container or virtual environment.

When using custom command:

- **Hostname and port must match** the values set in the Port and Hostname fields above
- You **must include `--cors app://obsidian.md`** to allow Obsidian to embed the OpenCode interface

Example:
```bash
opencode serve --port 14096 --hostname 127.0.0.1 --cors app://obsidian.md
```

Other settings (port, hostname, auto-start, view location, context injection) are available through the settings UI and are self-explanatory.

### Context injection (experimental)

This plugin can automatically inject context to the running OC instance: list of open notes and currently selected text.

Currently, this is work-in-progress feature with some limitations - it won't work when creating new session from OC interface.

## Windows Troubleshooting

If you see "Executable not found at 'opencode'" despite opencode being installed:

1. Find your opencode.cmd path:
   ```
   where opencode.cmd
   ```

2. Configure the full path in plugin settings:
   ```
   C:\Users\{username}\AppData\Roaming\npm\opencode.cmd
   ```

This is due to Electron/Obsidian not fully inheriting PATH on Windows.

## Fork Bug Fixes

This fork (moyang-creator/opencode-obsidian) includes the following fixes on top of v0.2.1:

### Fix 1: HTTP header encoding error with non-ASCII vault paths

**Bug**: `request()` passes `this.projectDirectory` (full vault path) as raw `x-opencode-directory` header value. When vault path contains Chinese or other non-ASCII characters, browser `fetch()` throws `TypeError: String contains non ISO-8859-1 code point` and the request is never sent.

**Fix**: `encodeURIComponent(this.projectDirectory)` at `main.js:1226`.

### Fix 2: OPENCODE_SERVER_PASSWORD env var pollution

**Bug**: `spawn()` passes `{ ...process.env }` to child process, inheriting system env var `OPENCODE_SERVER_PASSWORD`. If set, opencode serve starts with HTTP Basic Auth (401), causing plugin health check to fail.

**Fix**: Override both `OPENCODE_SERVER_PASSWORD: ""` and `OPENCODE_SERVER_USERNAME: ""` in spawn env at `main.js:958` (custom command path) and `main.js:970` (built-in path).

### Fix 3: Custom command must use opencode.ps1 via PowerShell

**Root Cause**: With `shell: true`, `child_process.spawn` uses cmd.exe which cannot execute `.ps1` files. Also, the built-in port/hostname settings in the plugin settings panel may not properly pass `--cors app://obsidian.md` to the opencode server. Using `useCustomCommand: true` with the full command ensures all required flags are passed correctly.

Note: `opencode serve` is NOT a singleton — multiple instances can run simultaneously on different ports, each as an independent OS process. The `.ps1` wrapper is needed for proper parameter passing and working directory isolation, not to circumvent a singleton limitation.

**Fix**: Set `useCustomCommand: true` with:
```
powershell.exe -NoProfile -NonInteractive -Command "& opencode.ps1 serve --port ... --hostname 127.0.0.1 --cors app://obsidian.md"
```

### Fix 4: Port reverts to default 14096 when loadData() returns null

**Bug**: `loadSettings()` uses `Object.assign({}, DEFAULT_SETTINGS, await this.loadData())`. When Obsidian's `loadData()` returns `null/undefined` (race condition on startup), `Object.assign` ignores it and the `port` field falls back to `DEFAULT_SETTINGS.port = 14096`. The plugin then stubbornly uses port 14096 despite the correct config in data.json.

**Fix**: After `loadSettings()`, extract the actual port from `customCommand` using regex `/--port\s+(\d+)/` and force-override `this.settings.port` at `main.js:1520-1525`.

### Fix 5: Base64 path contains `/` breaks the iframe URL

**Bug**: `getUrl()` base64-encodes the project directory and puts it in the iframe URL path. Standard base64 contains `/` characters, which split the URL into multiple path segments. The opencode SPA cannot extract the project directory from the broken URL and shows a blank screen.

**Fix**: Wrap the base64 output with `encodeURIComponent()` at `main.js:938` to encode `/` as `%2F`, keeping the path as a single URL segment.

### Quick Install

Copy the 4 files (`main.js`, `data.json`, `manifest.json`, `styles.css`) to your vault's `.obsidian/plugins/opencode-obsidian/` directory, then reload Obsidian.

