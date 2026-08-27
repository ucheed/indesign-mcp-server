# Running the InDesign MCP Server on Windows

The server drives Adobe InDesign through its scripting `DoScript` entry point. The
*transport* to that entry point is OS-specific:

- **macOS** — AppleScript (`osascript`)
- **Windows** — COM automation (a generated VBScript run with `cscript`)

`executeInDesignScript()` picks the right transport automatically from
`process.platform`. The 40+ ExtendScript tool payloads are identical on both platforms.

## Prerequisites

- Windows 10/11
- Adobe InDesign installed **and running** before you use the server
- Node.js ≥ 18

## Install

```powershell
cd indesign-mcp-server
npm install
```

Claude Desktop config (`%APPDATA%\Claude\claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "indesign": {
      "command": "node",
      "args": ["C:\\path\\to\\indesign-mcp-server\\index.js"],
      "env": {}
    }
  }
}
```

Claude Code:

```powershell
claude mcp add indesign -- node C:\path\to\indesign-mcp-server\index.js
```

Start InDesign, then start the MCP client. Verify with a simple request (e.g. "create a
new A4 document in InDesign").

## Configuration (env vars)

Both transports target a specific InDesign install. Override via env if the defaults
don't match your machine:

| Env var | Platform | Default | When to change |
|---|---|---|---|
| `INDESIGN_APP_NAME` | macOS | `Adobe InDesign 2025` | Different InDesign year, e.g. `Adobe InDesign 2024` |
| `INDESIGN_PROGID` | Windows | `InDesign.Application` | Pin a version when several are installed, e.g. `InDesign.Application.2025` |

Set them in the MCP client config `env` block.

## Verified vs. not

- ✅ **Code / syntax** — `node --check index.js` passes; the macOS path is unchanged from
  the original.
- ⚠️ **Windows COM path — NOT yet run against a real InDesign install.** It needs one
  validation pass on a Windows machine with InDesign. Three things to confirm there:
  1. **ProgID.** `GetObject(, "InDesign.Application")` attaches to the running instance.
     If it grabs the wrong version, set `INDESIGN_PROGID` to a version-specific ProgID
     (e.g. `InDesign.Application.2025`).
  2. **DoScript language constant.** `1246973031` is `idScriptLanguage.javascript`,
     stable across versions — confirm scripts execute (not "language not supported").
  3. **Output encoding.** Results are written UTF-16LE and read back as `utf16le`. If
     returned text is garbled, that's the knob.

## How the Windows bridge works

`executeWindowsInDesignScript()`:
1. Writes the wrapped ExtendScript to `temp_script.jsx` (same as macOS).
2. Generates `temp_run.vbs` that reads the `.jsx`, attaches to InDesign via COM
   (`GetObject`, falling back to `CreateObject`), calls `app.DoScript(code, 1246973031)`,
   and writes the result to `temp_out.txt`.
3. Runs it with `cscript //nologo`, reads `temp_out.txt` back as UTF-16LE, and cleans up
   the temp files.

Reading the result from a file rather than `cscript` stdout avoids Windows console
codepage/encoding problems.
