# Windows support — how the port works

The original project (`lucdesign/indesign-mcp-server`) drives InDesign through
**AppleScript** (`osascript`), which only exists on macOS. This fork adds a
**Windows transport** and selects the right one automatically at runtime
(`process.platform`). The 36 tools (ExtendScript) are unchanged and identical
on both platforms.

## What was changed

One method in `index.js`, `executeInDesignScript()`, was split into two transports:

**macOS — `runViaAppleScript()`** (original behaviour):
```js
tell application "Adobe InDesign 2025"
  do script POSIX file "…temp_script.jsx" language javascript
end tell   // via osascript
```

**Windows — `runViaWindowsCOM()`** (new): a temporary VBScript is generated, which

1. reads the `.jsx` file as UTF-8 (`ADODB.Stream`),
2. connects to InDesign through COM — `CreateObject("InDesign.Application")`
   (version-independent ProgID, with fallback to `InDesign.Application.2026/2025/2024`),
3. runs the script via `app.DoScript(src, 1246973031)`
   (`1246973031` = `ScriptLanguage.JAVASCRIPT`),
4. writes the result back as UTF-8 (accents and non-ASCII text preserved).

Node then runs `cscript //nologo //B` and reads the output file.

## Prerequisites (Windows)
- Windows 10/11 with **Adobe InDesign** installed (tested on the v21 / "2026" release).
- **Node.js 18+**.
- InDesign preferably **running** before use (COM will otherwise try to start it).

## MCP configuration
`.mcp.json` at your project root (Claude Code), or the equivalent block in
`claude_desktop_config.json` (Claude Desktop):
```json
{ "mcpServers": { "indesign": {
  "command": "node",
  "args": ["C:/path/to/indesign-mcp-server/index.js"]
} } }
```
Global scope with the Claude Code CLI:
```
claude mcp add indesign -s user -- node C:/path/to/indesign-mcp-server/index.js
```

## Optional environment variables
- `INDESIGN_PROGID` — force a specific COM ProgID, e.g. `InDesign.Application.2026`.
- `INDESIGN_APP_NAME` (macOS) — application name used by AppleScript.

## Quick test
```
node index.js         # should print: Complete InDesign MCP server running on stdio
```
