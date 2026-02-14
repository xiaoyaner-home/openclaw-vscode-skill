# VS Code / Cursor Node Skill

Use this skill when interacting with code through a VS Code/Cursor IDE connected as an OpenClaw Node.

## Prerequisites

- The VS Code extension `openclaw-node-vscode` must be installed and connected (status bar 🟢)
- Check node availability: `nodes describe --node "<node-name>"`

## Usage Pattern

All commands use `nodes invoke`:

```
nodes invoke --node "<node-name>" --invokeCommand "<command>" --invokeParamsJson '{"key":"value"}'
```

## Commands Reference

### 📁 File Operations

#### `vscode.file.read`
Read file content with optional line range.
```json
{ "path": "src/main.ts", "offset": 0, "limit": 100 }
→ { "content": "...", "totalLines": 250, "language": "typescript" }
```

#### `vscode.file.write`
Create or overwrite a file.
```json
{ "path": "src/new.ts", "content": "export const x = 1;\n" }
→ { "ok": true, "created": true }
```

#### `vscode.file.edit`
Precise text replacement (like the Edit tool).
```json
{ "path": "src/main.ts", "oldText": "const x = 1;", "newText": "const x = 2;" }
→ { "ok": true, "replacements": 1 }
```

#### `vscode.file.delete`
Delete file (default: move to trash).
```json
{ "path": "src/unused.ts", "useTrash": true }
→ { "ok": true }
```

### 📂 Directory

#### `vscode.dir.list`
List directory entries or search with glob.
```json
{ "path": "src", "pattern": "**/*.ts" }
→ { "entries": [{ "name": "main.ts", "type": "file", "size": 1234 }] }
```

### 🧠 Language Intelligence

#### `vscode.lang.definition`
Go to definition at position.
```json
{ "path": "src/main.ts", "line": 10, "character": 5 }
→ { "locations": [{ "path": "src/utils.ts", "line": 42, "character": 0 }] }
```

#### `vscode.lang.references`
Find all references.
```json
{ "path": "src/main.ts", "line": 10, "character": 5, "includeDeclaration": true }
→ { "locations": [{ "path": "...", "line": ..., "character": ... }] }
```

#### `vscode.lang.hover`
Get type information at position.
```json
{ "path": "src/main.ts", "line": 10, "character": 5 }
→ { "contents": ["```typescript\nconst x: number\n```"] }
```

#### `vscode.lang.symbols`
File symbols (with path) or workspace symbol search (with query).
```json
{ "path": "src/main.ts" }
→ { "symbols": [{ "name": "myFunc", "kind": "Function", "path": "src/main.ts", "line": 5, "endLine": 20 }] }
```
```json
{ "query": "handleRequest" }
→ { "symbols": [...] }  // workspace-wide search
```

#### `vscode.lang.rename`
Safe cross-file rename.
```json
{ "path": "src/main.ts", "line": 10, "character": 5, "newName": "newVarName" }
→ { "ok": true, "filesChanged": 3, "editsApplied": 7 }
```

#### `vscode.lang.codeActions`
List available quick fixes / refactors at a range.
```json
{ "path": "src/main.ts", "line": 10, "kind": "quickfix" }
→ { "actions": [{ "title": "Add missing import", "kind": "quickfix", "isPreferred": true, "index": 0 }] }
```

#### `vscode.lang.applyCodeAction`
Apply a code action by index (from codeActions result).
```json
{ "path": "src/main.ts", "line": 10, "kind": "quickfix", "index": 0 }
→ { "ok": true, "title": "Add missing import" }
```

### ⚡ Code Operations

#### `vscode.code.format`
Format document with configured formatter (Prettier etc).
```json
{ "path": "src/main.ts" }
→ { "ok": true, "editsApplied": 12 }
```

### 👁️ Editor Context

#### `vscode.editor.active`
Get currently focused file info.
```json
{}
→ { "path": "src/main.ts", "language": "typescript", "selections": [{ "line": 10, "character": 0 }] }
```

#### `vscode.editor.openFiles`
All open editor tabs.
```json
{}
→ { "files": [{ "path": "src/main.ts", "language": "typescript", "isDirty": false }] }
```

#### `vscode.editor.selections`
Get selected text in active editor.
```json
{}
→ { "path": "src/main.ts", "selections": [{ "startLine": 5, "endLine": 10, "text": "..." }] }
```

#### `vscode.diagnostics.get`
Get TypeScript/ESLint errors.
```json
{ "path": "src/main.ts" }
→ { "diagnostics": [{ "path": "...", "line": 10, "severity": "error", "message": "...", "source": "ts" }] }
```

#### `vscode.workspace.info`
Workspace name and root path.
```json
{}
→ { "name": "joy-desk", "rootPath": "/Users/dsj/Codes/joy-desk", "folders": [...] }
```

### 🧪 Testing

#### `vscode.test.list`
Check if test framework is available.
```json
{}
→ { "available": true, "message": "Test framework detected..." }
```

#### `vscode.test.run`
Run tests (all, by file, or debug mode).
```json
{ "path": "src/main.test.ts" }
→ { "ok": true, "message": "Test run triggered." }
```
```json
{ "debug": true }
→ runs tests in debug mode
```

#### `vscode.test.results`
Check latest test results via diagnostics.
```json
{}
→ { "hasResults": true, "summary": "No test errors detected." }
```

### 🔀 Git

#### `vscode.git.status`
Branch, staged/modified/untracked files, ahead/behind.
```json
{}
→ { "branch": "main", "staged": [], "modified": ["src/main.ts"], "untracked": [], "ahead": 2, "behind": 0 }
```

#### `vscode.git.diff`
Get diff (optionally staged, or vs a ref).
```json
{ "staged": true }
→ { "diff": "..." }
```
```json
{ "path": "src/main.ts", "ref": "HEAD~1" }
→ { "diff": "..." }
```

#### `vscode.git.log`
Commit history.
```json
{ "limit": 10, "oneline": true }
→ { "log": "abc1234 fix bug\ndef5678 add feature\n..." }
```

#### `vscode.git.blame`
Line-level blame.
```json
{ "path": "src/main.ts", "startLine": 10, "endLine": 20 }
→ { "blame": "..." }  // porcelain format
```

#### `vscode.git.stage`
Stage files.
```json
{ "paths": ["src/main.ts", "src/utils.ts"] }
→ { "ok": true }
```

#### `vscode.git.unstage`
Unstage files.
```json
{ "paths": ["src/main.ts"] }
→ { "ok": true }
```

#### `vscode.git.commit`
Commit staged changes.
```json
{ "message": "fix: resolve type error", "amend": false }
→ { "ok": true, "output": "..." }
```

#### `vscode.git.stash`
Stash operations.
```json
{ "action": "push", "message": "WIP" }
{ "action": "pop" }
{ "action": "list" }
→ { "ok": true, "output": "..." }
```

### 🐛 Debug

#### `vscode.debug.launch`
Start a debug session.
```json
{ "name": "Launch Server" }
→ uses named config from launch.json
```
```json
{ "config": { "type": "node", "request": "launch", "name": "Debug", "program": "${workspaceFolder}/src/index.ts" } }
→ uses inline config
```

#### `vscode.debug.stop`
Stop active debug session.
```json
{}
→ { "ok": true }
```

#### `vscode.debug.breakpoint`
Manage breakpoints.
```json
{ "action": "add", "path": "src/main.ts", "line": 42, "condition": "x > 10" }
{ "action": "remove", "path": "src/main.ts", "line": 42 }
{ "action": "list" }
→ { "ok": true, "breakpoints": [...] }
{ "action": "clear" }
```

#### `vscode.debug.evaluate`
Evaluate expression in debug context.
```json
{ "expression": "myVar.length", "context": "repl" }
→ { "result": "42", "type": "number" }
```

#### `vscode.debug.stackTrace`
Get call stack.
```json
{ "levels": 10 }
→ { "frames": [{ "id": 0, "name": "handleRequest", "path": "src/main.ts", "line": 42 }] }
```

#### `vscode.debug.variables`
Inspect variables at current breakpoint.
```json
{ "scope": "locals" }
→ { "variables": [{ "name": "x", "value": "42", "type": "number", "variablesReference": 0 }] }
```

#### `vscode.debug.status`
Check if debug session is active.
```json
{}
→ { "active": true, "sessionName": "Launch Server", "sessionType": "node" }
```

### 🖥️ Terminal (optional, disabled by default)

#### `vscode.terminal.run`
Execute whitelisted command.
```json
{ "command": "pnpm test", "cwd": "packages/core", "timeoutMs": 60000 }
→ { "exitCode": 0, "stdout": "...", "stderr": "...", "timedOut": false }
```

## Typical Workflows

### Fix a type error
1. `vscode.diagnostics.get` → find errors
2. `vscode.lang.hover` → understand types
3. `vscode.lang.definition` → find source
4. `vscode.file.edit` → fix the code
5. `vscode.diagnostics.get` → verify fix
6. `vscode.code.format` → format

### Safe refactor
1. `vscode.lang.references` → find all usages
2. `vscode.lang.rename` → rename across files
3. `vscode.diagnostics.get` → check no breakage
4. `vscode.test.run` → run tests
5. `vscode.git.diff` → review changes
6. `vscode.git.stage` + `vscode.git.commit`

### Debug a failing test
1. `vscode.test.run { "path": "..." }` → run test
2. `vscode.diagnostics.get` → check errors
3. `vscode.debug.breakpoint { "action": "add", ... }` → set breakpoint
4. `vscode.test.run { "debug": true }` → debug test
5. `vscode.debug.variables` → inspect state
6. `vscode.debug.evaluate { "expression": "..." }` → evaluate
