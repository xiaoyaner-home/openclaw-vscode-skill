---
name: vscode-node
description: Operate on code through a VS Code/Cursor IDE connected as an OpenClaw Node. Provides 40+ commands for file operations, language intelligence, git, testing, debugging, and Cursor Agent CLI integration. Use when you need to read/write/edit code, navigate definitions/references, run tests, debug, or delegate coding tasks to Cursor Agent.
metadata:
  {"openclaw": {"requires": {"tools": ["nodes"]}}}
---

# VS Code / Cursor Node Skill

Control a VS Code or Cursor IDE remotely through the OpenClaw Node protocol.

## When to Use This Skill

- Reading, writing, or editing code files in a VS Code/Cursor workspace
- Navigating code (go to definition, find references, hover info)
- Running git operations (status, diff, commit)
- Running or debugging tests
- Delegating complex coding tasks to Cursor Agent CLI
- Getting diagnostics (TypeScript errors, ESLint warnings)

## Prerequisites

1. The VS Code extension `openclaw-node-vscode` must be **installed and connected** in the target IDE
2. The node must appear in `nodes status` with a green status
3. Commands must be in the Gateway's `allowCommands` whitelist

### Check Node Availability

```
nodes status
```

Look for the node name (e.g., `my-vscode`) in the output. If not listed, the extension is not connected.

### Get Node Details

```
nodes describe --node "<node-name>"
```

This shows available commands, connection time, and device info.

## Command Invocation Pattern

**All commands** use the `nodes invoke` tool with these parameters:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `node` | ✅ | Node name (as shown in `nodes status`) |
| `invokeCommand` | ✅ | Command name (e.g., `vscode.file.read`) |
| `invokeParamsJson` | ❌ | JSON string of command parameters |
| `invokeTimeoutMs` | ❌ | Gateway internal timeout (default: 90000ms) |
| `timeoutMs` | ❌ | HTTP layer timeout (should be > invokeTimeoutMs) |

### Example

```
nodes invoke
  --node "my-vscode"
  --invokeCommand "vscode.file.read"
  --invokeParamsJson '{"path": "src/main.ts"}'
  --invokeTimeoutMs 30000
  --timeoutMs 35000
```

### Important: Timeout Configuration

For long-running commands (git operations, agent tasks, large file searches), you **must** set both timeouts:

- `invokeTimeoutMs`: How long the Gateway waits for the extension to respond
- `timeoutMs`: How long the HTTP request waits (must be **larger** than `invokeTimeoutMs`)

**Recommended timeouts by command type:**

| Command Type | invokeTimeoutMs | timeoutMs |
|-------------|----------------|-----------|
| File read/write | 15000 | 20000 |
| Language intelligence | 15000 | 20000 |
| Git operations | 30000 | 35000 |
| Test run | 60000 | 65000 |
| Agent run (plan/ask) | 180000 | 185000 |
| Agent run (agent mode) | 300000 | 305000 |

---

## Command Reference

### 📁 File Operations

#### `vscode.file.read` — Read file content

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | Relative path from workspace root |
| `offset` | number | ❌ | Start line (0-indexed, default: 0) |
| `limit` | number | ❌ | Max lines to read |

**Returns:** `{ content: string, totalLines: number, language: string }`

```json
{"path": "src/main.ts", "offset": 0, "limit": 100}
→ {"content": "import ...\n...", "totalLines": 250, "language": "typescript"}
```

#### `vscode.file.write` — Create or overwrite a file

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | Relative path from workspace root |
| `content` | string | ✅ | Full file content to write |

**Returns:** `{ ok: true, created: boolean }`

```json
{"path": "src/new.ts", "content": "export const x = 1;\n"}
→ {"ok": true, "created": true}
```

**Note:** If `openclaw.readOnly` is enabled in the extension, write operations will be rejected. If `openclaw.confirmWrites` is enabled, the user will see a confirmation dialog.

#### `vscode.file.edit` — Precise text replacement

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | Relative path from workspace root |
| `oldText` | string | ✅ | Exact text to find (must match exactly) |
| `newText` | string | ✅ | Replacement text |

**Returns:** `{ ok: true, replacements: number }`

```json
{"path": "src/main.ts", "oldText": "const x = 1;", "newText": "const x = 2;"}
→ {"ok": true, "replacements": 1}
```

**Important:** `oldText` must match exactly including whitespace and newlines. If no match is found, returns `replacements: 0`.

#### `vscode.file.delete` — Delete a file

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | Relative path from workspace root |
| `useTrash` | boolean | ❌ | Move to trash instead of permanent delete (default: true) |

**Returns:** `{ ok: true }`

### 📂 Directory

#### `vscode.dir.list` — List directory contents

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ❌ | Directory path (default: workspace root) |
| `pattern` | string | ❌ | Glob pattern to filter (e.g., `**/*.ts`) |
| `recursive` | boolean | ❌ | Include subdirectories (default: false) |

**Returns:** `{ entries: [{ name, type, size }] }`

```json
{"path": "src", "pattern": "**/*.ts", "recursive": true}
→ {"entries": [{"name": "main.ts", "type": "file", "size": 1234}, ...]}
```

### 🧠 Language Intelligence

These commands leverage the IDE's language server (TypeScript, ESLint, etc.) for intelligent code navigation.

#### `vscode.lang.definition` — Go to definition

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |
| `line` | number | ✅ | Line number (0-indexed) |
| `character` | number | ✅ | Column number (0-indexed) |

**Returns:** `{ locations: [{ path, line, character }] }`

#### `vscode.lang.references` — Find all references

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |
| `line` | number | ✅ | Line number (0-indexed) |
| `character` | number | ✅ | Column number (0-indexed) |
| `includeDeclaration` | boolean | ❌ | Include the declaration itself (default: true) |

**Returns:** `{ locations: [{ path, line, character }] }`

#### `vscode.lang.hover` — Get type/documentation info

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |
| `line` | number | ✅ | Line number (0-indexed) |
| `character` | number | ✅ | Column number (0-indexed) |

**Returns:** `{ contents: [string] }` — Markdown-formatted type info and documentation

#### `vscode.lang.symbols` — Search symbols

**Mode 1: Document symbols** (provide `path`)

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |

**Mode 2: Workspace symbol search** (provide `query`)

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | ✅ | Search query |

**Returns:** `{ symbols: [{ name, kind, path, line, endLine }] }`

**Note:** Workspace symbol search may return many results. The extension paginates at 200 symbols per response.

#### `vscode.lang.rename` — Rename symbol across files

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |
| `line` | number | ✅ | Line number (0-indexed) |
| `character` | number | ✅ | Column number (0-indexed) |
| `newName` | string | ✅ | New name for the symbol |

**Returns:** `{ ok: true, filesChanged: number, editsApplied: number }`

**Important:** This performs a real cross-file rename using the language server. It is safe and respects TypeScript/language semantics.

#### `vscode.lang.codeActions` — List available quick fixes

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |
| `line` | number | ✅ | Line number |
| `kind` | string | ❌ | Filter by kind: `quickfix`, `refactor`, `source` |

**Returns:** `{ actions: [{ title, kind, isPreferred, index }] }`

#### `vscode.lang.applyCodeAction` — Apply a code action

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |
| `line` | number | ✅ | Line number |
| `kind` | string | ❌ | Filter kind |
| `index` | number | ✅ | Index from `codeActions` result |

**Returns:** `{ ok: true, title: string }`

#### `vscode.code.format` — Format document

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |

**Returns:** `{ ok: true, editsApplied: number }`

### 👁️ Editor Context

#### `vscode.editor.active` — Get active file info

No parameters required.

**Returns:** `{ path, language, selections: [{ line, character }] }`

#### `vscode.editor.openFiles` — List all open tabs

No parameters required.

**Returns:** `{ files: [{ path, language, isDirty }] }`

#### `vscode.editor.selections` — Get selected text

No parameters required.

**Returns:** `{ path, selections: [{ startLine, endLine, text }] }`

### ⚠️ Diagnostics

#### `vscode.diagnostics.get` — Get errors and warnings

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ❌ | Filter to specific file (default: all open files) |
| `severity` | string | ❌ | Filter: `error`, `warning`, `info`, `hint` |

**Returns:** `{ diagnostics: [{ path, line, severity, message, source }] }`

### 🌿 Git

#### `vscode.git.status` — Working tree status

No parameters required.

**Returns:** `{ branch, staged: [], modified: [], untracked: [], ahead, behind }`

#### `vscode.git.diff` — View diffs

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ❌ | Specific file (default: all) |
| `staged` | boolean | ❌ | Show staged diff (default: false) |
| `ref` | string | ❌ | Compare against ref (e.g., `HEAD~1`) |

**Returns:** `{ diff: string }`

#### `vscode.git.log` — Commit history

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `limit` | number | ❌ | Max commits (default: 10) |
| `oneline` | boolean | ❌ | One-line format |
| `path` | string | ❌ | Filter by file path |

**Returns:** `{ log: string }`

#### `vscode.git.blame` — Line-level blame

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ✅ | File path |
| `startLine` | number | ❌ | Start line |
| `endLine` | number | ❌ | End line |

**Returns:** `{ blame: string }` (porcelain format)

#### `vscode.git.stage` — Stage files

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `paths` | string[] | ✅ | Array of file paths to stage |

**Returns:** `{ ok: true }`

#### `vscode.git.unstage` — Unstage files

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `paths` | string[] | ✅ | Array of file paths to unstage |

**Returns:** `{ ok: true }`

#### `vscode.git.commit` — Commit staged changes

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `message` | string | ✅ | Commit message |
| `amend` | boolean | ❌ | Amend last commit (default: false) |

**Returns:** `{ ok: true, output: string }`

#### `vscode.git.stash` — Stash operations

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | ✅ | One of: `push`, `pop`, `list`, `drop` |
| `message` | string | ❌ | Stash message (for `push`) |

**Returns:** `{ ok: true, output: string }`

### 🧪 Testing

#### `vscode.test.list` — Check test framework

No parameters required.

**Returns:** `{ available: boolean, message: string }`

#### `vscode.test.run` — Run tests

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | ❌ | Run tests in specific file |
| `debug` | boolean | ❌ | Run in debug mode |

**Returns:** `{ ok: true, message: string }`

#### `vscode.test.results` — Get latest results

No parameters required.

**Returns:** `{ hasResults: boolean, summary: string }`

### 🐛 Debug

#### `vscode.debug.launch` — Start debug session

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | ❌ | Named config from launch.json |
| `config` | object | ❌ | Inline debug configuration |

Provide either `name` (to use a launch.json config) or `config` (inline).

#### `vscode.debug.stop` — Stop debugging

No parameters required.

#### `vscode.debug.breakpoint` — Manage breakpoints

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | ✅ | `add`, `remove`, `list`, or `clear` |
| `path` | string | ❌ | File path (for add/remove) |
| `line` | number | ❌ | Line number (for add/remove) |
| `condition` | string | ❌ | Conditional expression (for add) |

#### `vscode.debug.evaluate` — Evaluate expression

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `expression` | string | ✅ | Expression to evaluate |
| `context` | string | ❌ | `repl`, `watch`, or `hover` |

**Returns:** `{ result: string, type: string }`

**Note:** Only works when stopped at a breakpoint during an active debug session.

#### `vscode.debug.stackTrace` — Get call stack

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `levels` | number | ❌ | Max stack frames (default: 20) |

**Returns:** `{ frames: [{ id, name, path, line }] }`

#### `vscode.debug.variables` — Inspect variables

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `scope` | string | ❌ | `locals`, `globals`, or `closure` |

**Returns:** `{ variables: [{ name, value, type, variablesReference }] }`

#### `vscode.debug.status` — Check debug session

No parameters required.

**Returns:** `{ active: boolean, sessionName?: string, sessionType?: string }`

### 🖥️ Terminal (disabled by default)

Terminal commands are **disabled by default** for security. They must be explicitly enabled in extension settings, and only whitelisted commands are allowed.

#### `vscode.terminal.run` — Execute command

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `command` | string | ✅ | Command to execute (must be in whitelist) |
| `cwd` | string | ❌ | Working directory |
| `timeoutMs` | number | ❌ | Timeout in ms (default: 30000) |

**Returns:** `{ exitCode: number, stdout: string, stderr: string, timedOut: boolean }`

### 🤖 Cursor Agent CLI (Cursor IDE only)

These commands integrate with the Cursor Agent CLI for AI-assisted coding. Only available when the IDE is Cursor (not VS Code).

#### `vscode.agent.status` — Check CLI availability

No parameters required.

**Returns:** `{ available: boolean, version?: string, authenticated?: boolean }`

#### `vscode.agent.run` — Run Cursor Agent

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | ✅ | Task description or question |
| `mode` | string | ❌ | `agent` (default), `plan`, or `ask` |
| `model` | string | ❌ | Model name (default: extension setting) |
| `cwd` | string | ❌ | Working directory |
| `timeoutMs` | number | ❌ | Timeout (default: 300000ms) |

**Mode descriptions:**
- **agent**: Full access — reads and writes files, executes changes
- **plan**: Analysis only — proposes changes without executing them
- **ask**: Read-only Q&A — answers questions about the codebase

**Returns:** `{ output: string, exitCode: number }`

**Important:** Agent mode can modify files. Use `plan` mode first to review proposed changes, then `agent` mode to execute.

#### `vscode.agent.setup` — Open setup wizard

No parameters required. Opens the extension's setup wizard in the IDE.

### 📦 Workspace

#### `vscode.workspace.info` — Workspace information

No parameters required.

**Returns:** `{ name: string, rootPath: string, folders: string[] }`

---

## Common Workflows

### Fix a Type Error

```
Step 1: Find errors
  → vscode.diagnostics.get
  
Step 2: Understand the type at error location
  → vscode.lang.hover {"path": "src/main.ts", "line": 10, "character": 5}
  
Step 3: Find where the type is defined
  → vscode.lang.definition {"path": "src/main.ts", "line": 10, "character": 5}
  
Step 4: Fix the code
  → vscode.file.edit {"path": "src/main.ts", "oldText": "...", "newText": "..."}
  
Step 5: Verify fix
  → vscode.diagnostics.get
  
Step 6: Format
  → vscode.code.format {"path": "src/main.ts"}
```

### Safe Cross-File Refactor

```
Step 1: Find all usages of the symbol
  → vscode.lang.references {"path": "src/main.ts", "line": 10, "character": 5}
  
Step 2: Rename using language server (safe, cross-file)
  → vscode.lang.rename {"path": "src/main.ts", "line": 10, "character": 5, "newName": "newName"}
  
Step 3: Check for breakage
  → vscode.diagnostics.get
  
Step 4: Run tests
  → vscode.test.run
  
Step 5: Review changes
  → vscode.git.diff
  
Step 6: Commit
  → vscode.git.stage {"paths": ["src/main.ts", "src/utils.ts"]}
  → vscode.git.commit {"message": "refactor: rename X to Y"}
```

### Delegate Complex Task to Cursor Agent

```
Step 1: Understand current state
  → vscode.workspace.info
  → vscode.git.status
  
Step 2: Plan first (read-only analysis)
  → vscode.agent.run {"prompt": "Analyze how to add error handling to all API endpoints", "mode": "plan"}
  (use invokeTimeoutMs: 180000, timeoutMs: 185000)
  
Step 3: Review the plan output, then execute
  → vscode.agent.run {"prompt": "Add try-catch error handling to all API endpoints in src/routes/", "mode": "agent"}
  (use invokeTimeoutMs: 300000, timeoutMs: 305000)
  
Step 4: Review what changed
  → vscode.git.diff
  → vscode.diagnostics.get
```

---

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| `node command not allowed` | Command not in Gateway whitelist | Add to `gateway.nodes.allowCommands` config |
| `node not found` | Extension not connected | Check extension status bar, reconnect |
| `timeout` | Operation took too long | Increase `invokeTimeoutMs` and `timeoutMs` |
| `path traversal blocked` | Path outside workspace | Use relative paths from workspace root |
| `read-only mode` | Extension in read-only mode | Disable `openclaw.readOnly` in extension settings |
| `terminal disabled` | Terminal not enabled | Enable `openclaw.terminal.enabled` and add to whitelist |

## Security Notes

- All file paths are **relative to the workspace root**. Absolute paths and `../` traversal are blocked.
- Write operations respect the extension's `readOnly` and `confirmWrites` settings.
- Terminal commands are disabled by default and require explicit whitelist configuration.
- The Gateway can further restrict allowed commands via `gateway.nodes.allowCommands`.
- Each extension instance has a unique device identity (Ed25519 keypair) that must be approved by the Gateway.
