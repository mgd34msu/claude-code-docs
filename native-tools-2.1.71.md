# Claude Code v2.1.71 — Native Tools Reference

**Version:** Claude Code v2.1.71  
**Source:** `cli.js` (612,918 lines, minified/bundled)  
**Date:** 2026-03-07  
**Document:** Part 1 of 3

---

## Table of Contents (All 3 Parts)

### Part 1 (This Document)
1. [Overview & Tool Summary Table](#1-overview--tool-summary-table)
2. [Tool Infrastructure](#2-tool-infrastructure)
   - 2.1 [Tool Registry & Tool List](#21-tool-registry--tool-list)
   - 2.2 [Tool Categorization Arrays](#22-tool-categorization-arrays)
   - 2.3 [Tool Aliases](#23-tool-aliases)
   - 2.4 [Tool Resolution Function `Fi()`](#24-tool-resolution-function-fi)
   - 2.5 [Permission System](#25-permission-system)
   - 2.6 [Concurrency Model](#26-concurrency-model)
   - 2.7 [Tool Result Formatting Protocol](#27-tool-result-formatting-protocol)
   - 2.8 [Tool Prompt/Description System](#28-tool-promptdescription-system)
   - 2.9 [Permission Mode Interaction](#29-permission-mode-interaction)
   - 2.10 [Key Variable Name Glossary](#210-key-variable-name-glossary)
3. [Core File I/O Tools](#3-core-file-io-tools)
   - 3.1 [Read (`u4`)](#31-read-u4)
   - 3.2 [Write (`Y3`)](#32-write-y3)
   - 3.3 [Edit (`Yq`)](#33-edit-yq)
   - 3.4 [Glob (`zz`)](#34-glob-zz)
   - 3.5 [Grep (`fY`)](#35-grep-fy)
   - 3.6 [NotebookEdit (`NM`)](#36-notebookedit-nm)
4. [Execution Tool](#4-execution-tool)
   - 4.1 [Bash (`Hq`)](#41-bash-hq)

### Part 2
5. Web Tools
   - 5.1 WebFetch (`UP` / `VM`)
   - 5.2 WebSearch (`PS1` / `tV`)
6. Agent & Task Tools
   - 6.1 Agent (`Tq`)
   - 6.2 TaskCreate (`US`)
   - 6.3 TaskGet (`c16`)
   - 6.4 TaskUpdate (`KL`)
   - 6.5 TaskList (`l16`)
   - 6.6 TaskOutput (`SI`)
   - 6.7 TaskStop (`RI`)
7. User Interaction & Planning
   - 7.1 AskUserQuestion (`b_`)
   - 7.2 EnterPlanMode (`p16`)
   - 7.3 ExitPlanMode

### Part 3
8. Extensibility Tools
   - 8.1 Skill (`nj`)
   - 8.2 ToolSearch (`OW`)
   - 8.3 TodoWrite (`HF`)
   - 8.4 LSP
   - 8.5 EnterWorktree (`FT1`)
9. Scheduling Tools
   - 9.1 CronCreate (`HU`)
   - 9.2 CronDelete (`H_6`)
   - 9.3 CronList (`RS1`)
10. Team Communication
    - 10.1 SendMessage (`Gx`)
11. Browser & Computer Use Tools (18 tools)
12. Appendix: Constants, Tool Aliases, Variable Names

---

## 1. Overview & Tool Summary Table

Claude Code v2.1.71 ships approximately 45 native tools compiled into a single `cli.js` bundle. Tools are organized into categories based on their function, permission model, and availability constraints.

### Tool Summary Table

| # | Tool Name | Variable | Category | `isReadOnly` | `isConcurrencySafe` | `shouldDefer` | `strict` | `maxResultSizeChars` | Feature Flag / Condition |
|---|-----------|----------|----------|:---:|:---:|:---:|:---:|:---:|---|
| 1 | **Read** | `u4` | File I/O | ✓ | ✓ | — | ✓ | 100,000 | Always enabled |
| 2 | **Write** | `Y3` | File I/O | ✗ | ✗ | — | ✓ | 100,000 | Always enabled |
| 3 | **Edit** | `Yq` | File I/O | ✗ | ✗ | — | ✓ | 100,000 | Always enabled |
| 4 | **Glob** | `zz` | File I/O | ✓ | ✓ | — | — | 100,000 | Always enabled |
| 5 | **Grep** | `fY` | File I/O | ✓ | ✓ | — | ✓ | 20,000 | Always enabled |
| 6 | **NotebookEdit** | `NM` | File I/O | ✗ | ✗ | ✓ | — | 100,000 | Always enabled |
| 7 | **Bash** | `Hq` (`f4`) | Execution | Contextual | = `isReadOnly` | — | ✓ | 30,000 | Always enabled |
| 8 | **WebFetch** | `UP` (`VM`) | Web | ✓ | ✓ | ✓ | — | 100,000 | Always enabled |
| 9 | **WebSearch** | `PS1` (`tV`) | Web | ✓ | ✓ | ✓ | — | 100,000 | firstParty/foundry always; Vertex: claude-4 family only |
| 10 | **Agent** | `Tq` | Agent/Task | — | — | — | — | — | Internal; excluded from model tool list (`cT6`) |
| 11 | **TaskCreate** | `US` | Agent/Task | — | — | — | — | — | Async+Teams |
| 12 | **TaskGet** | `c16` | Agent/Task | — | — | — | — | — | Async+Teams |
| 13 | **TaskUpdate** | `KL` | Agent/Task | — | — | — | — | — | Async+Teams |
| 14 | **TaskList** | `l16` | Agent/Task | — | — | — | — | — | Async+Teams |
| 15 | **TaskOutput** | `SI` | Agent/Task | — | — | — | — | — | Internal; excluded from model tool list (`cT6`) |
| 16 | **TaskStop** | `RI` | Agent/Task | — | — | — | — | — | Internal; excluded (`cT6`); alias: `KillShell` |
| 17 | **AskUserQuestion** | `b_` | Interaction | — | — | — | — | — | Internal; excluded from model tool list (`cT6`) |
| 18 | **EnterPlanMode** | `p16` | Planning | — | — | — | — | — | Internal; excluded from model tool list (`cT6`) |
| 19 | **ExitPlanMode** | `aM` | Planning | — | — | — | — | — | Kept in plan mode only |
| 20 | **Skill** | `nj` | Extensibility | — | — | — | — | — | Async-safe (`QT1`) |
| 21 | **ToolSearch** | `OW` | Extensibility | — | — | — | — | — | Async-safe (`QT1`) |
| 22 | **TodoWrite** | `HF` | Extensibility | — | — | — | — | — | Async-safe (`QT1`) |
| 23 | **LSP** | — | Extensibility | — | — | — | — | — | — |
| 24 | **EnterWorktree** | `FT1` | Extensibility | — | — | — | — | — | Async-safe (`QT1`) |
| 25 | **StructuredOutput** | `KX` | Internal | — | — | — | — | — | Async-safe; excluded from pP() by default |
| 26 | **CronCreate** | `HU` | Scheduling | ✗ | ✗ | ✓ | — | 100,000 | `tengu_kairos_cron` feature gate (disabled by default) |
| 27 | **CronDelete** | `H_6` | Scheduling | ✗ | ✗ | ✓ | — | — | `tengu_kairos_cron` feature gate |
| 28 | **CronList** | `RS1` | Scheduling | ✓ | ✓ | ✓ | — | — | `tengu_kairos_cron` feature gate |
| 29 | **SendMessage** | `Gx` | Team Comm | Contextual | ✗ | ✓ | — | 100,000 | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` + `tengu_amber_flint` |
| 30–47 | **Browser/Computer Use** (18) | various | Browser | — | — | — | — | — | Conditional on mode/platform |

**Notes:**
- `strict: true` means the tool enforces strict JSON schema validation on inputs.
- `shouldDefer: true` means the tool is loaded lazily and appears in `<available-deferred-tools>` in agent system prompts.
- `maxResultSizeChars` is the hard cap on tool result content returned to the model.
- Tools marked `—` for a property inherit defaults or the property is not applicable.
- Dash (`—`) in Feature Flag column means always enabled with no additional gating.

---

## 2. Tool Infrastructure

### 2.1 Tool Registry & Tool List

All tools are registered via the `YU()` function (line ~451550 in `cli.js`), which returns the master tool array. This is the source of truth for what tools exist:

```javascript
function YU() {
  return [
    dT6,       // (internal tool)
    XS1,       // (internal tool)
    Hq,        // Bash
    ...(cH() ? [] : [zU, bu]),  // Glob (zU) and Grep (bu) — excluded when cH() mode is active
    HX,        // (internal tool)
    KY,        // Read
    dP,        // Edit
    gP,        // Write
    ln,        // NotebookEdit
    UP,        // WebFetch
    pN,        // (internal tool)
    PS1,       // WebSearch
    MS1,       // (internal tool)
    ev6,       // (internal tool)
    jA6,       // (internal tool)
    Ka6,       // (internal tool)
    ...(iH() ? [$Oq, WOq, EOq, uOq] : []),  // simple/headless-mode-only tools
    ...(FHq ? [FHq] : []),
    ...(QHq ? [QHq] : []),
    tc8,
    ...(Yk6() ? [F$q] : []),
    ...(Z7() ? [vzz(), kzz(), Ezz()] : []),  // Team tools: TeamCreate, TeamDelete, SendMessage
    ...(gHq ? [gHq] : []),
    ...(uHq ? [uHq] : []),
    ...Nzz,    // Cron tools: [CronCreateTool, CronDeleteTool, CronListTool]
    ...(BHq ? [BHq] : []),
    ...(mHq ? [mHq] : []),
    ...(pHq?.() ? [pHq()] : []),
    ...(UHq ? [UHq] : []),
    an,
    tn,
    ...(CS1 ? [CS1] : []),
    ...(hS1 ? [hS1] : []),
    ...(IS1 ? [IS1] : []),
    ...(bS1 ? [bS1] : []),
    ...(Wx() ? [wd6] : []),
  ];
}
```

**Effective tool list `pP(A)`** — the filtered set actually provided to the model:

```javascript
pP = (A) => {
  // SIMPLE mode: only Bash, Read, Edit (3 tools)
  if ($1(process.env.CLAUDE_CODE_SIMPLE)) return Jk6([Hq, KY, dP], A);

  // Exclude internal/conditional tools by name
  let excludedNames = new Set([
    an.name, tn.name,
    ...CS1, hS1, IS1, bS1,
    KX,   // "StructuredOutput"
  ]);
  let allTools = YU().filter((w) => !excludedNames.has(w.name));
  let filtered = Jk6(allTools, A);  // remove blocked tools

  // REPL mode: filter tools overlapping with pV1 set
  if ($1(process.env.CLAUDE_REPL_MODE)) {
    if (filtered.some((_) => R5(_, pV1))) filtered = filtered.filter((_) => !QC4.has(_.name));
  }

  // Apply isEnabled() gate
  let enabled = filtered.map((w) => w.isEnabled());
  return filtered.filter((w, _) => enabled[_]);
};
```

**External tool injection `HA6(A, q)`** — merges built-in tools with external (MCP) tools:

```javascript
function HA6(A, q) {
  let K = pP(A);            // base enabled built-in tools
  if ($1(process.env.CLAUDE_CODE_SIMPLE)) return K;
  let Y = Jk6(q, A);       // filter external tools by blocked rules
  return zW([...K, ...Y], "name"); // deduplicate by name
}
```

**`CLAUDE_CODE_SIMPLE` mode** reduces the entire toolset to 3 tools: `Bash`, `Read`, `Edit`.

---

### 2.2 Tool Categorization Arrays

Several named arrays and sets control how tools are categorized for permission checks, file pattern matching, and scheduling:

**`dd` — Bash tool set** (line ~89111):
```javascript
dd = [f4, BP3].filter((A) => A != null);
// f4 = "Bash", BP3 = null (platform-specific variant, null on Linux/macOS)
// Result: dd = ["Bash"]
```

**`gP3` — Read-only/search tool names** (line ~89213):
```javascript
gP3 = [...dd, zz, fY, u4, VM, tV];
// = ["Bash", "Glob", "Grep", "Read", "WebFetch", "WebSearch"]
```
Used for permission-prompting category grouping (safe/read-only operations).

**`FP3` — Write tool names** (line ~89213):
```javascript
FP3 = [Yq, Y3, NM];
// = ["Edit", "Write", "NotebookEdit"]
```
Used for permission-prompting category grouping (write operations).

**`CjY` — Write-tracking tools** (line 288840):
```javascript
CjY = ["Edit", "Write", "NotebookEdit"]
```
Used by `pC8(name)` to identify which tools trigger write-decision telemetry.

**`Y68.filePatternTools`** (line ~64988):
```javascript
filePatternTools: ["Read", "Write", "Edit", "Glob", "NotebookRead", "NotebookEdit"]
```
These tools support file-pattern permission rules (e.g., `allowedTools("Edit(src/**)")` syntax).

**`Y68.bashPrefixTools`**:
```javascript
bashPrefixTools: ["Bash"]
```
Bash supports prefix-based permission rules (e.g., `allowedTools("Bash(git *)")`).

**`Y68.customValidation`**:
```javascript
customValidation: {
  WebSearch: (A) => {
    if (A.includes("*") || A.includes("?"))
      return { valid: false, error: "WebSearch does not support wildcards", ... };
    return { valid: true };
  }
}
```

**`wL_` — Search/read tools set** (line ~89219):
```javascript
wL_ = new Set([u4, ...dd, fY, zz, tV, VM, Yq, Y3, ...[]]
// = new Set(["Read", "Bash", "Grep", "Glob", "WebSearch", "WebFetch", "Edit", "Write"])
```

**`cT6` — Always-excluded tools** (line ~290087):
```javascript
cT6 = new Set([SI, aM, p16, Tq, b_, RI]);
// SI = "TaskOutput"
// aM = internal plan-mode tool (ExitPlanMode)
// p16 = "EnterPlanMode"
// Tq = "Agent"
// b_ = "AskUserQuestion"
// RI = "TaskStop"
```
These tools are **always excluded** from tool lists shown to the model. Exception: `aM` is kept in `"plan"` mode.

**`QT1` — Async-safe tools** (tools available during background/async agent operations):
```javascript
QT1 = new Set([
  u4,    // "Read"
  tV,    // "WebSearch"
  HF,    // "TodoWrite"
  fY,    // "Grep"
  VM,    // "WebFetch"
  zz,    // "Glob"
  ...dd, // "Bash"
  Yq,    // "Edit"
  Y3,    // "Write"
  NM,    // "NotebookEdit"
  nj,    // "Skill"
  KX,    // "StructuredOutput"
  OW,    // "ToolSearch"
  FT1,   // "EnterWorktree"
])
```

**`Gf4` — Additional async tools when team features active** (`Z7() && AW()`):
```javascript
Gf4 = new Set([
  US,   // "TaskCreate"
  c16,  // "TaskGet"
  l16,  // "TaskList"
  KL,   // "TaskUpdate"
  Gx,   // "SendMessage"
])
```

---

### 2.3 Tool Aliases

The alias table `aUA` maps deprecated or alternate tool names to canonical internal names:

```javascript
aUA = {
  Task: Tq,              // "Task" → "Agent"
  KillShell: RI,          // "KillShell" → "TaskStop"
  AgentOutputTool: SI,    // "AgentOutputTool" → "TaskOutput"
  BashOutputTool: SI,     // "BashOutputTool" → "TaskOutput"
}
```

**Resolution functions:**
- `bf(name)` → `aUA[name] ?? name` — forward lookup (alias → canonical)
- `sUA(canonical)` → reverse lookup (canonical → all aliases)
- `Sj(ruleString)` uses `bf()` to resolve aliases in permission rule strings

**MCP tool naming:** `K68(serverName, toolName)` → `"mcp__{serverName}__{toolName}"` with non-alphanumeric characters replaced by `_`. `ok(name)` parses MCP tool names back into `{ serverName, toolName }`.

---

### 2.4 Tool Resolution Function `Fi()`

`Fi()` resolves a list of tool-name strings (from API request or config) into actual tool objects, applying all filtering rules:

```javascript
function Fi(A, q, K = false, Y = false) {
  let { tools: z, disallowedTools: w, source: _, permissionMode: $ } = A;

  // Step 1: Filter tools by async/builtin mode (Y=true bypasses filtering)
  let O = Y ? q : Ah8({ tools: q, isBuiltIn: _ === "built-in", isAsync: K, permissionMode: $ });

  // Step 2: Build disallowed set from disallowedTools rules
  let H = new Set(w?.map((G) => Sj(G).toolName) ?? []);

  // Step 3: Filter out disallowed
  let j = O.filter((G) => !H.has(G.name));

  // Step 4: Wildcard check — undefined or ["*"] allows all tools
  if (z === undefined || (z.length === 1 && z[0] === "*"))
    return { hasWildcard: true, validTools: [], invalidTools: [], resolvedTools: j };

  // Step 5: Resolve named tools
  let M = new Map(j.map(G => [G.name, G]));
  let D = [], X = [], P = [], W = new Set(), Z;

  for (let G of z) {
    let { toolName: f, ruleContent: V } = Sj(G);
    if (f === Tq) {  // "Agent" tool — extract allowedAgentTypes
      if (V) Z = V.split(",").map(v => v.trim());
      if (!Y) { D.push(G); continue; }
    }
    let N = M.get(f);
    if (N) {
      if (D.push(G), !W.has(N)) { P.push(N); W.add(N); }
    } else X.push(G);
  }

  return {
    hasWildcard: false,
    validTools: D,           // rule strings that matched
    invalidTools: X,         // rule strings that didn't match (unknown tools)
    resolvedTools: P,        // actual tool objects
    allowedAgentTypes: Z,    // parsed from "Agent(type1,type2)" rule
  };
}
```

**`Ah8()` — context-based tool filter:**
```javascript
function Ah8({ tools, isBuiltIn, isAsync = false, permissionMode }) {
  return tools.filter((z) => {
    if (z.name.startsWith("mcp__")) return true;           // MCP tools always pass through
    if (R5(z, aM) && permissionMode === "plan") return true; // ExitPlanMode kept in plan mode
    if (cT6.has(z.name)) return false;                      // always-excluded tools
    if (!isBuiltIn && eC8.has(z.name)) return false;        // non-builtin excluded tools
    if (isAsync && !QT1.has(z.name)) {                      // async mode: only QT1 tools
      if (Z7() && AW()) {                                   // + team tools if teams active
        if (R5(z, Tq)) return true;
        if (Gf4.has(z.name)) return true;
      }
      return false;
    }
    return true;
  });
}
```

**Blocked-tool filter `Jk6(A, q)`:**
```javascript
function Jk6(A, q) {
  let K = JU(q);  // get disallowed tool rules from permission context
  return A.filter((Y) => {
    let z = JI6(Y);  // get canonical name
    return !K.some((w) => w.ruleValue.toolName === z && w.ruleValue.ruleContent === void 0);
    // Only blocks exact tool name match with no pattern qualifier
  });
}
```

---

### 2.5 Permission System

**Permission context structure** — the `toolPermissionContext` object:

```typescript
{
  mode: "default" | "plan" | "bypassPermissions",
  alwaysAllowRules: {
    command: Rule[],
    localSettings: Rule[],
    userSettings: Rule[],
    projectSettings: Rule[],
    session: Rule[],
    cliArg: Rule[],
  },
  alwaysDenyRules:  { /* same structure */ },
  alwaysAskRules:   { /* same structure */ },
  additionalWorkingDirectories: Map<string, { path: string, source: string }>,
}
```

**Permission behaviors:** `"allow"` | `"deny"` | `"ask"`

Each tool's `checkPermissions()` returns:
```typescript
{
  behavior: "allow" | "deny" | "ask" | "passthrough",
  message?: string,
  decisionReason?: {
    type: "rule" | "mode" | "workingDir" | "other",
    rule?: Rule,
    mode?: string,
    reason?: string,
  },
  suggestions?: Array<{ type: "addRules", rules: Rule[], behavior: string, destination: string }>,
  updatedInput?: object,
}
```

**Permission update types** (applied via `nz(context, update)`):
- `setMode` — changes permission mode
- `addRules` — adds allow/deny/ask rules to a destination
- `replaceRules` — replaces all rules for a destination/behavior
- `addDirectories` — adds allowed working directories
- `removeRules` — removes specific rules
- `removeDirectories` — removes working directories

**File-path permission check flow `U66(tool, input, context)`:**
1. Check `getPath()` method exists on input (else ask)
2. Resolve full absolute path
3. UNC path guard (`\\` or `//` prefix) → ask
4. Suspicious Windows path patterns → ask
5. `ZP(path, context, "read", "deny")` matched → deny
6. `ZP(path, context, "read", "ask")` matched → ask
7. Edit permission rules check `a26()` → allow/deny
8. `Mx(path, context)` — is path inside a working directory → allow
9. Worktree check `EQ8()` → allow/deny/passthrough
10. `ZP(path, context, "read", "allow")` matched → allow
11. Default → ask ("path is outside allowed working directories")

**`ZP(path, context, operation, behavior)`** — gitignore-style pattern matcher:
- Builds rule set via `Lbq(context, operation, behavior)`
- Uses `ignore` library for pattern matching
- Computes relative path from working dir via `Vbq()`
- Handles `/**` suffix patterns
- Returns matching rule object or `null`

**`shouldDefer` flag:** When `true`, the tool's permission dialog is shown while the tool runs rather than blocking execution. CronCreate, CronDelete, CronList, NotebookEdit, WebFetch, WebSearch, SendMessage, and all Scheduling tools have `shouldDefer: true`.

---

### 2.6 Concurrency Model

**`isConcurrencySafe(input?)` protocol:**
- Returns `true` → tool can run concurrently alongside other tool calls
- Returns `false` → tool must run exclusively; parallelism is blocked

| Concurrency-Safe (can parallelize) | Not Safe (must run serially) |
|---|---|
| Read, Glob, Grep, WebFetch, WebSearch | Write, Edit, NotebookEdit |
| Bash (only when `isReadOnly` is true) | Bash (when writing/mutating) |
| CronList | CronCreate, CronDelete |
| SendMessage `type=message` or `broadcast` | SendMessage other types |

**Async mode tool set (`QT1`):** When an Agent sub-task runs asynchronously (`isAsync: true`), only tools in `QT1` are available. This prevents background agents from spawning new agents or interacting with UI-bound tools. Team tools (`TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`, `SendMessage`) become available additionally when `Z7() && AW()` (teams feature enabled and user is an active agent member).

**Concurrency dispatch** (line 250244): At the tool dispatch site, `isConcurrencySafe()` is called on pending tool calls. Only `true`-returning tools are dispatched in parallel with currently-running tool calls.

---

### 2.7 Tool Result Formatting Protocol

Every tool implements `mapToolResultToToolResultBlockParam(data, toolUseId)`, which converts the tool's output into an Anthropic API `tool_result` content block:

```javascript
// Standard text result:
{ tool_use_id: q, type: "tool_result", content: "text string" }

// Error result:
{ tool_use_id: q, type: "tool_result", content: "Error message", is_error: true }

// Rich content (images, structured blocks):
{ tool_use_id: q, type: "tool_result", content: [{ type: "text", text: "..." }, { type: "image", ... }] }
```

**Render functions on each tool (for CLI display):**
- `renderToolUseMessage(input)` → `string | null` — "Tool: X" prefix line
- `renderToolUseProgressMessage(input)` → `string | null` — shown during execution
- `renderToolUseRejectedMessage()` → React element — shown when user denies
- `renderToolUseErrorMessage(result, {verbose})` → React element — shown on error
- `renderToolResultMessage(data, toolUseId, {verbose})` → React element — shown after completion

**Tool-to-API serialization for MCP tools** (via `Pn8()`):
```javascript
{
  name: Z.name,
  description: await Z.prompt({ getToolPermissionContext, tools, agents }),
  input_schema: Z.inputJSONSchema ?? {},
}
```
MCP tools use `inputJSONSchema` (raw JSON Schema object). Native built-in tools pass their description and schema through the Anthropic client SDK's built-in mechanism.

**`maxResultSizeChars`:** Hard cap on the size of tool result content returned to the model. Grep has a lower limit (20,000); Bash has 30,000; most other tools use 100,000.

---

### 2.8 Tool Prompt/Description System

Each tool has two text fields:
- `async description()` → short one-line description shown in tool listings
- `async prompt()` → detailed multi-paragraph prompt injected into the system prompt at session start

These are concatenated into the model's system prompt when the session starts and when the active tool set changes. The `prompt()` method receives `{ getToolPermissionContext, tools, agents }` so it can reference the current tool set.

---

### 2.9 Permission Mode Interaction

| Mode | Behavior |
|------|---------|
| `"default"` | Standard permission checks apply; user prompted for write/network ops |
| `"plan"` | Write tools filtered out; `aM` (ExitPlanMode) kept despite being in `cT6` |
| `"bypassPermissions"` | `x46()` returns true; path checks in `U66()` return allow without prompting |
| `CLAUDE_CODE_SIMPLE` env | Only 3 tools: Bash, Read, Edit |
| `CLAUDE_REPL_MODE` env | Filters out tools overlapping with `pV1` set |

---

### 2.10 Key Variable Name Glossary

Minified variable names from `cli.js` mapped to tool names:

```
Tq   = "Agent"
RI   = "TaskStop"           (alias: "KillShell")
SI   = "TaskOutput"         (aliases: "AgentOutputTool", "BashOutputTool")
p16  = "EnterPlanMode"
b_   = "AskUserQuestion"
aM   = ExitPlanMode (internal plan-mode tool)
f4   = "Bash"
u4   = "Read"
VM   = "WebFetch"
Yq   = "Edit"
Y3   = "Write"
zz   = "Glob"
fY   = "Grep"
NM   = "NotebookEdit"
tV   = "WebSearch"
HF   = "TodoWrite"
nj   = "Skill"
KX   = "StructuredOutput"
OW   = "ToolSearch"
FT1  = "EnterWorktree"
US   = "TaskCreate"
c16  = "TaskGet"
l16  = "TaskList"
KL   = "TaskUpdate"
Gx   = "SendMessage"
HU   = "CronCreate"
H_6  = "CronDelete"
RS1  = "CronList"
Aw   = "team-lead"          (constant string)
nN   = "claude-swarm"       (constant)
```

**Tool object groupings (from file-io.md):**
```
gP3 = [...dd, zz, fY, u4, VM, tV]   // read-only/search tools
FP3 = [Yq, Y3, NM]                  // write tools
dd  = [f4]                           // = ["Bash"]
```
(Lines 89111–89213 in `cli.js`)

---

## 3. Core File I/O Tools

### 3.1 Read (`u4`)

**Internal names:** Tool object: `u4` (line 88021). Tool name constant: `u4 = "Read"`. Input schema function: `$AY` (line 249666). Output schema: `OAY` (line 249686). Tool object defined at line 249756.

**searchHint:** `"read file contents"`  
**isReadOnly():** `true`  
**isConcurrencySafe():** `true`  
**isSearchOrReadCommand():** `{ isSearch: false, isRead: true }`  
**maxResultSizeChars:** 100,000  
**strict:** `true`

#### Input Schema (`I.strictObject`)

```typescript
{
  file_path: string
    // "The absolute path to the file to read"

  offset?: number
    // "The line number to start reading from.
    //  Only provide if the file is too large to read at once"

  limit?: number
    // "The number of lines to read.
    //  Only provide if the file is too large to read at once."

  pages?: string
    // `Page range for PDF files (e.g., "1-5", "3", "10-20").
    //  Only applicable to PDF files. Maximum 20 pages per request.`
    // (Constant: fG6 = 20, line 234541)
}
```

**Input parameter aliases:** `filePath → file_path`, `filepath → file_path`, `path → file_path`

#### Output Schema (`I.discriminatedUnion("type", [...])`)

Discriminated union on the `type` field:

**`type: "text"`**
```typescript
{
  file: {
    filePath: string      // "The path to the file that was read"
    content: string       // "The content of the file"
    numLines: number      // "Number of lines in the returned content"
    startLine: number     // "The starting line number"
    totalLines: number    // "Total number of lines in the file"
  }
}
```

**`type: "image"`**
```typescript
{
  file: {
    base64: string        // "Base64-encoded image data"
    type: "image/jpeg" | "image/png" | "image/gif" | "image/webp"
    originalSize: number  // "Original file size in bytes"
    dimensions?: {
      originalWidth?: number    // "Original image width in pixels"
      originalHeight?: number   // "Original image height in pixels"
      displayWidth?: number     // "Displayed image width in pixels (after resizing)"
      displayHeight?: number    // "Displayed image height in pixels (after resizing)"
    }
  }
}
```

**`type: "notebook"`**
```typescript
{
  file: {
    filePath: string  // "The path to the notebook file"
    cells: any[]      // "Array of notebook cells"
  }
}
```

**`type: "pdf"`**
```typescript
{
  file: {
    filePath: string      // "The path to the PDF file"
    base64: string        // "Base64-encoded PDF data"
    originalSize: number  // "Original file size in bytes"
  }
}
```

**`type: "parts"`** (large PDF split into page images)
```typescript
{
  file: {
    filePath: string      // "The path to the PDF file"
    originalSize: number  // "Original file size in bytes"
    count: number         // "Number of pages extracted"
    outputDir: string     // "Directory containing extracted page images"
  }
}
```

#### Constants & Limits

| Constant | Symbol | Value | cli.js Line |
|----------|--------|-------|-------------|
| Default line read limit | `Jx6` | 2,000 lines | 88022 |
| Max line character width | `$X3` | 2,000 chars (truncated beyond) | 88023 |
| Max token output | `wAY` | 25,000 tokens | 249598 |
| Max token output env override | `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | overrides `wAY` | — |
| Max file size (default) | `uE8` | 262,144 bytes (256 KB) | 547400 |
| Max PDF pages per request | `fG6` | 20 | 234541 |
| Supported image extensions | `P24` | `new Set(["png", "jpg", "jpeg", "gif", "webp"])` | 249665 |
| maxResultSizeChars | — | 100,000 | — |

**Blocked device files** (hard-coded deny list):
`/dev/null`, `/dev/zero`, `/dev/urandom`, `/dev/full`, `/dev/stdin`, `/dev/tty`, `/dev/console`, `/dev/stdout`, `/dev/stderr`, `/dev/fd/0`, `/dev/fd/1`, `/dev/fd/2`

#### Permission Model

- `isReadOnly() → true`
- `isConcurrencySafe() → true`
- Permission check: `U66(KY, A, K.toolPermissionContext)` — checks `"read"` operation against deny list
- Validates: file not in denied directories (`ZP(Y, ..., "read", "deny")`), file not binary (unless image/PDF), not a blocking device file

#### Key Implementation Details

- **Call signature:** `call({ file_path, offset=1, limit=undefined, pages }, { readFileState, fileReadingLimits })`
- Routes to `X24()` based on file extension: text, image, `.ipynb` (notebook), PDF
- PDF reading requires firstParty mode (`jx6()` function, line 88042)
- On ENOENT, tries symlink/alternate path resolution (`YAY`), then fuzzy path suggestion (`p66`)
- Jupyter notebook files (`.ipynb`) are parsed and returned as structured cells (not raw JSON)
- Dynamic skill dir triggers are fired on read (line 249874)
- `CLAUDE_CODE_SIMPLE` env var skips skill trigger logic

---

### 3.2 Write (`Y3`)

**Internal names:** Tool name constant: `Y3 = "Write"` (line 89148). Input schema function: `h4z` (line 426603). Output schema: `I4z` (line 426611). Tool object at line 426642.

**isReadOnly():** `false`  
**isConcurrencySafe():** `false`  
**maxResultSizeChars:** 100,000  
**strict:** `true`

#### Input Schema (`I.strictObject`)

```typescript
{
  file_path: string
    // "The absolute path to the file to write (must be absolute, not relative)"

  content: string
    // "The content to write to the file"
}
```

**Input parameter aliases:** `filePath → file_path`, `filepath → file_path`, `path → file_path`

#### Output Schema (`I.object`)

```typescript
{
  type: "create" | "update"   // "Whether a new file was created or an existing file was updated"
  filePath: string             // "The path to the file that was written"
  content: string              // "The content that was written to the file"
  structuredPatch: Array<{     // "Diff patch showing the changes"
    oldStart: number
    oldLines: number
    newStart: number
    newLines: number
    lines: string[]
  }>                           // (type: pp8)
  originalFile: string | null  // "The original file content before the write (null for new files)"
  gitDiff?: {                  // optional; only when CLAUDE_CODE_REMOTE env + tengu_quartz_lantern flag
    filename: string
    status: "modified" | "added"
    additions: number
    deletions: number
    changes: number
    patch: string
    repository?: string | null  // "GitHub owner/repo when available"
  }
}
```

#### Constants & Limits

- **`maxResultSizeChars`:** 100,000
- **`strict: true`**
- No separate line/character limits on content written

#### Permission Model

- `isReadOnly() → false`
- `isConcurrencySafe() → false`
- Permission check: `a26(gP, A, K.toolPermissionContext)` — checks `"edit"` permission
- `toAutoClassifierInput(A)` → `"${file_path}: ${content}"`

#### Validation Logic

1. Validates file path (`XR1`) and content
2. Checks file not in denied directory (`ZP(..., "edit", "deny")`)
3. If file exists: requires it was previously read in session (`readFileState.get(path)` must exist) — **error code 2**
4. If file was externally modified since last read: **error code 3** ("File has been modified since read")
5. New files (ENOENT): allowed immediately — **error code 0** for not found

#### Key Implementation Details

1. Normalize path (`t4`)
2. Check dynamic skill dir triggers
3. Call `Gi.beforeFileEdited()` hook
4. Validate read state and modification timestamp (throws `Hx6` if stale)
5. Read original content for diff generation
6. Create parent directories (`H.mkdirSync(O)`)
7. Write file via `ZA6()` preserving original encoding (UTF-8 vs UTF-16LE detected via BOM) and line endings
8. Notify LSP server: `dn().changeFile()` + `dn().saveFile()`
9. Update `readFileState` with new content and timestamp
10. If file is `CLAUDE.md`: emit telemetry event `tengu_write_claudemd`
11. Optionally compute git diff (`NR1()`) when `CLAUDE_CODE_REMOTE` env + `tengu_quartz_lantern` feature flag

---

### 3.3 Edit (`Yq`)

**Internal names:** Referenced at line 525977. Input schema function: `PR1` (line 424899). Output schema: `H9q` (line 424921). Alternate schema: `j9q` (line 424952). Tool object at line 525976.

**isReadOnly():** `false`  
**isConcurrencySafe():** `false`  
**maxResultSizeChars:** 100,000  
**strict:** `true`

#### Primary Input Schema (`PR1` — `I.strictObject`)

```typescript
{
  file_path: string
    // "The absolute path to the file to modify"

  old_string: string
    // "The text to replace"

  new_string: string
    // "The text to replace it with (must be different from old_string)"

  replace_all?: boolean
    // "Replace all occurrences of old_string (default false)"
    // default: false
}
```

**Input parameter aliases:** `old_str → old_string`, `new_str → new_string`, `oldString → old_string`, `newString → new_string`, `filePath → file_path`, `filepath → file_path`, `path → file_path`

#### Alternate Input Schema (`j9q` — `I.object`) — Line-reference-based edits

This alternate schema is defined at line 424952 but the tool object at line 525996 uses `PR1()` for `inputSchema`. `j9q` may be an internal/experimental variant.

```typescript
{
  file_path: string
    // "The absolute path to the file to modify"

  edits: Array<
    | { set: {
        ref: string       // 'Line reference "LINE#HASH"'
        body: string[]    // "Replacement lines (empty array to delete the line)"
      }}
    | { set_range: {
        beg: string       // 'Start line reference "LINE#HASH"'
        end: string       // 'End line reference "LINE#HASH"'
        body: string[]    // "Replacement lines (empty array to delete the range)"
      }}
    | { insert: {
        before?: string   // 'Insert before this line "LINE#HASH"'
        after?: string    // 'Insert after this line "LINE#HASH"'
        body: string[]    // "Lines to insert (must be non-empty)"
      }}
    | { replace: {
        old_text: string  // "Text to find"
        new_text: string  // "Replacement text"
        all?: boolean     // "Replace all occurrences"
      }}
  >
    // "Array of edit operations"
}
```

#### Output Schema (`H9q` — `I.object`)

```typescript
{
  filePath: string        // "The file path that was edited"
  oldString: string       // "The original string that was replaced"
  newString: string       // "The new string that replaced it"
  originalFile: string    // "The original file contents before editing"
  structuredPatch: Array<{  // "Diff patch showing the changes" (type: pp8)
    oldStart: number
    oldLines: number
    newStart: number
    newLines: number
    lines: string[]
  }>
  userModified: boolean   // "Whether the user modified the proposed changes"
  replaceAll: boolean     // "Whether all occurrences were replaced"
  gitDiff?: {             // optional; same conditions as Write tool
    filename: string
    status: "modified" | "added"
    additions: number
    deletions: number
    changes: number
    patch: string
    repository?: string | null
  }
}
```

#### Constants & Limits

- **`maxResultSizeChars`:** 100,000
- **`strict: true`**

#### Permission Model

- `isReadOnly() → false`
- `isConcurrencySafe() → false`
- Permission check: `a26(dP, A, K.toolPermissionContext)` — checks `"edit"` permission

#### Validation Error Codes (line 526032)

| Code | Condition | Behavior |
|------|-----------|----------|
| 1 | `old_string === new_string` | ask |
| 2 | File in denied directory | ask |
| 3 | File exists with content + `old_string === ""` | ask ("Cannot create new file — file already exists") |
| 4 | File not found + `old_string !== ""` | ask |
| 5 | File is `.ipynb` | ask (use NotebookEdit instead) |
| 6 | File not in `readFileState` | ask ("File has not been read yet") |
| 7 | File modified since read (unless byte-identical) | ask |
| 8 | `old_string` not found in file | ask ("String to replace not found") |
| 9 | Multiple matches + `replace_all=false` | ask |

**Special `old_string = ""` behavior:** When `old_string` is empty string and file is empty or doesn't exist, creates the file (new file creation path).

#### Key Implementation Details

1. Normalize path; trigger skill dir hooks; call `beforeFileEdited()`
2. Read current file content (detect UTF-8 vs UTF-16LE via BOM)
3. Validate read state freshness
4. Fuzzy match `old_string` in file content via `P56()`
5. Apply whitespace normalization via `z06()` to `new_string`
6. Generate diff with `RO1()` → `{ patch, updatedFile }`
7. Write updated file via `ZA6()` preserving encoding and line endings
8. Notify LSP server (change + save)
9. Update `readFileState`
10. Track `CLAUDE.md` writes via telemetry
11. Optionally compute git diff (remote + feature flag)

**Read-before-write enforcement (shared with Write):** Both Write and Edit require the file to be in `readFileState` before modification of an existing file. Write uses error code 2, Edit uses error code 6. This is a session-level safety lock.

**Modification timestamp freshness (shared with Write):** Both tools check `mtimeMs` against the stored read timestamp. Write: error code 3, Edit: error code 7. Exception for Edit: if content is byte-identical to stored state, the stale check is bypassed.

---

### 3.4 Glob (`zz`)

**Internal names:** Tool name constant: `zz = "Glob"` (line 89113). Input schema function: `m4z` (line 427510). Output schema: `g4z` (line 427520). Tool object at line 427534.

**isReadOnly():** `true`  
**isConcurrencySafe():** `true`  
**isSearchOrReadCommand():** `{ isSearch: true, isRead: false }`  
**maxResultSizeChars:** 100,000  
**strict:** not set

#### Input Schema (`I.strictObject`)

```typescript
{
  pattern: string
    // "The glob pattern to match files against"

  path?: string
    // 'The directory to search in. If not specified, the current working directory will be used.
    //  IMPORTANT: Omit this field to use the default directory.
    //  DO NOT enter "undefined" or "null" — simply omit it for the default behavior.
    //  Must be a valid directory path if provided.'
}
```

**Input parameter aliases:** `directory → path`

#### Output Schema (`I.object`)

```typescript
{
  durationMs: number    // "Time taken to execute the search in milliseconds"
  numFiles: number      // "Total number of files found"
  filenames: string[]   // "Array of file paths that match the pattern"
  truncated: boolean    // "Whether results were truncated (limited to 100 files)"
}
```

#### Constants & Limits

- **Default result limit:** 100 files (`globLimits?.maxResults ?? 100`, line 427613)
- **Sort order:** By modification time descending (most recent first), alphabetical tiebreak
- **`maxResultSizeChars`:** 100,000

#### Permission Model

- `isReadOnly() → true`
- `isConcurrencySafe() → true`
- `isSearchOrReadCommand() → { isSearch: true, isRead: false }`
- Permission check: `U66(zU, A, K.toolPermissionContext)`

#### Validation Logic

- If `path` provided: must exist (stat), must be a directory (not a file) — **error code 2**
- Path not found → fuzzy path suggestion, **error code 1**
- UNC paths (`\\`) or double-slash paths: bypass validation

#### Key Implementation Details

- **Call signature:** `call(A, { abortController, getAppState, globLimits })`
- Internally calls `zYq(pattern, path, { limit: maxResults, offset: 0 }, signal, toolPermissionContext)` — built on the `glob` library
- Empty results → formatted as "No files found"
- If truncated → appends "(Results are truncated. Consider using a more specific path or pattern.)"
- Sort is by mtime descending — **different from standard lexicographic glob expansion**

---

### 3.5 Grep (`fY`)

**Internal names:** Tool name constant: `fY = "Grep"` (line 89133). Input schema function: `x4z` (line 427092). Output schema: `B4z` (line 427157). Tool object at line 427169.

**isReadOnly():** `true`  
**isConcurrencySafe():** `true`  
**isSearchOrReadCommand():** `{ isSearch: true, isRead: false }`  
**maxResultSizeChars:** 20,000 (lower than all other tools)  
**strict:** `true`

#### Input Schema (`I.strictObject`)

```typescript
{
  pattern: string
    // "The regular expression pattern to search for in file contents"

  path?: string
    // "File or directory to search in (rg PATH). Defaults to current working directory."

  glob?: string
    // 'Glob pattern to filter files (e.g. "*.js", "*.{ts,tsx}") — maps to rg --glob'

  output_mode?: "content" | "files_with_matches" | "count"
    // 'Output mode:
    //   "content" — shows matching lines (supports -A/-B/-C context, -n line numbers, head_limit)
    //   "files_with_matches" — shows file paths (supports head_limit)
    //   "count" — shows match counts (supports head_limit)
    //  Defaults to "files_with_matches".'

  "-B"?: number
    // 'Number of lines before each match (rg -B). Requires output_mode: "content".'

  "-A"?: number
    // 'Number of lines after each match (rg -A). Requires output_mode: "content".'

  "-C"?: number
    // "Alias for context."

  context?: number
    // 'Number of lines before and after each match (rg -C). Requires output_mode: "content".'

  "-n"?: boolean
    // 'Show line numbers in output (rg -n). Requires output_mode: "content". Defaults to true.'

  "-i"?: boolean
    // "Case insensitive search (rg -i)"

  type?: string
    // "File type to search (rg --type). Common types: js, py, rust, go, java, etc."

  head_limit?: number
    // 'Limit output to first N lines/entries (like "| head -N").
    //  Works across all output modes. Defaults to 0 (unlimited).'

  offset?: number
    // 'Skip first N entries before applying head_limit (like "| tail -n +N | head -N").
    //  Works across all output modes. Defaults to 0.'

  multiline?: boolean
    // "Enable multiline mode where . matches newlines (rg -U --multiline-dotall). Default: false."
}
```

**Input parameter aliases:** `c → -C`, `C → -C`, `a → -A`, `A → -A`, `b → -B`, `B → -B`, `n → -n`, `i → -i`, `include → glob`, `regex → pattern`, `search → pattern`, `directory → path`

#### Output Schema (`I.object`)

```typescript
{
  mode?: "content" | "files_with_matches" | "count"
  numFiles: number
  filenames: string[]
  content?: string
  numLines?: number
  numMatches?: number
  appliedLimit?: number
  appliedOffset?: number
}
```

#### Constants & Limits

| Constant | Symbol | Value | cli.js Line |
|----------|--------|-------|-------------|
| maxResultSizeChars | — | 20,000 | — |
| Max column width | — | 500 chars (`--max-columns 500`) | 427345 |
| Excluded VCS dirs | `u4z` | `[".git", ".svn", ".hg", ".bzr"]` | 427156 |
| Default `-n` (line numbers) | — | `true` in content mode | — |
| Default `-i` (case insensitive) | — | `false` | — |
| Default `offset` | — | `0` | — |
| Default `multiline` | — | `false` | — |
| Default `output_mode` | — | `"files_with_matches"` | — |

#### Permission Model

- `isReadOnly() → true`
- `isConcurrencySafe() → true`
- `isSearchOrReadCommand() → { isSearch: true, isRead: false }`
- Permission check: `U66(bu, A, K.toolPermissionContext)`
- Also respects tool permission context deny list for excluded paths

#### Key Implementation Details

Grip always uses `ripgrep` (`rg`) as the underlying search engine. Ripgrep args constructed as:

1. Base: `["--hidden", "--glob", "!.git", "--glob", "!.svn", "--glob", "!.hg", "--glob", "!.bzr", "--max-columns", "500"]`
2. Multiline: adds `["-U", "--multiline-dotall"]` if `multiline=true`
3. Case: adds `["-i"]` if `-i=true`
4. Mode flags: `-l` for `files_with_matches`, `-c` for `count`, nothing extra for `content`
5. Line numbers: adds `-n` if content mode and `-n=true`
6. Context flags: `-C`/`context` takes precedence over separate `-B`/`-A`
7. Patterns starting with `-`: prefix with `-e` (to avoid flag collision)
8. `type`: adds `--type <type>` if provided
9. `glob` parameter: splits on whitespace, expands brace patterns, adds `--glob` per entry
10. Permission-based excludes applied via `bv6()`
11. `.gitignore` excludes applied via `Uq6()`
12. Run ripgrep via `iy(args, path, signal)`
13. `offset`/`head_limit` slicing via `Ad8()`
14. `files_with_matches` results sorted by mtime descending, then alphabetically
15. Paths relativized to CWD via `qd8()`

---

### 3.6 NotebookEdit (`NM`)

**Internal names:** Tool name constant: `NM = "NotebookEdit"` (line 89152). Input schema function: `U4z` (line 427838). Output schema: `p4z` (line 427861). Tool object at line 427888.

**isReadOnly():** `false`  
**isConcurrencySafe():** `false`  
**shouldDefer:** `true` (line 427892) — loaded lazily; appears in `<available-deferred-tools>`  
**maxResultSizeChars:** 100,000  
**strict:** not set (uses `I.strictObject` for input but no `strict` flag on the tool object)

#### Input Schema (`I.strictObject`)

```typescript
{
  notebook_path: string
    // "The absolute path to the Jupyter notebook file to edit (must be absolute, not relative)"

  cell_id?: string
    // "The ID of the cell to edit.
    //  When inserting a new cell, the new cell will be inserted after the cell with this ID,
    //  or at the beginning if not specified."

  new_source: string
    // "The new source for the cell"

  cell_type?: "code" | "markdown"
    // "The type of the cell (code or markdown).
    //  If not specified, defaults to the current cell type.
    //  Required when using edit_mode=insert."

  edit_mode?: "replace" | "insert" | "delete"
    // "The type of edit to make (replace, insert, delete). Defaults to replace."
}
```

#### Output Schema (`I.object`)

```typescript
{
  new_source: string          // "The new source code written to the cell"
  cell_id?: string            // "The ID of the cell that was edited"
  cell_type: "code" | "markdown"  // "The type of the cell"
  language: string            // "The programming language of the notebook"
  edit_mode: string           // "The edit mode that was used"
  error?: string              // "Error message if the operation failed"
  notebook_path: string       // "The path to the notebook file"
  original_file: string       // "The original notebook content before modification"
  updated_file: string        // "The updated notebook content after modification"
}
```

#### Constants & Limits

- **`maxResultSizeChars`:** 100,000
- **`shouldDefer: true`**

#### Permission Model

- `isReadOnly() → false`
- `isConcurrencySafe() → false`
- Permission check: `a26(ln, A, K.toolPermissionContext)` — uses the same write-permission checker as Edit and Write

#### Validation Error Codes

| Code | Condition |
|------|-----------|
| 1 | File does not exist ("Notebook file does not exist") |
| 2 | File does not have `.ipynb` extension |
| 4 | `edit_mode` is not "replace", "insert", or "delete" |
| 5 | `edit_mode=insert` but `cell_type` not specified |
| 6 | File is not valid JSON |
| 7 | `cell_id` not specified when required, or numeric index not found |
| 8 | `cell_id` string not found in notebook |

#### Key Implementation Details

1. Resolve absolute path (if relative, join with CWD)
2. Call `_A6()` hook (agent mode only, `aw()`)
3. Read notebook file; detect encoding via `v0()`
4. Parse notebook JSON via `O8()`
5. Find cell by `cell_id` (`findIndex(f => f.id === K)`); fallback to numeric index via `ip6(K)`
6. For `insert`: increment index by 1 to insert after referenced cell
7. Auto-promote `replace` → `insert` if at end of cells array; auto-assign `cell_type="code"`
8. Detect language from `metadata.language_info.name` (defaults to `"python"`)
9. Cell ID assignment (nbformat >= 4.5): random UUID for insert, preserve original ID for replace
10. **delete:** `cells.splice(M, 1)`
11. **insert:** create cell object, `cells.splice(M, 0, newCell)`; reset `execution_count=null`, `outputs=[]` for code cells
12. **replace:** overwrite `source`; reset execution/outputs for code cells; optionally change `cell_type`
13. Serialize to JSON via `JSON.stringify(J, null, 1)` (1-space indent)
14. Write file via `ZA6()` preserving line endings
15. On any exception: returns `{ error: H.message }` rather than throwing — errors are contained in the `error` output field

**Cross-cutting note:** Edit tool returns error code 5 (`"use NotebookEdit instead"`) when `file_path` has a `.ipynb` extension, enforcing the tool boundary between Edit and NotebookEdit.

---

## 4. Execution Tool

### 4.1 Bash (`Hq`)

**Internal names:** Tool object: `Hq`. Tool name string: `f4 = "Bash"` (line implicit from `dd = [f4, BP3].filter(...)`). Input schema: `DCq` (full internal), `XCq` (public-facing). Output schema: `zSz`.

**searchHint:** `"execute shell commands"`  
**userFacingName:** `"Bash"` (or `"SandboxedBash"` when sandboxed and `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` env is set)  
**isEnabled():** Always `true`  
**isConcurrencySafe(A):** Delegates to `this.isReadOnly(A)` — safe only when read-only  
**maxResultSizeChars:** 30,000  
**strict:** `true`

#### Input Schema (`DCq` — `I.strictObject`)

The public-facing schema (`XCq`) omits `_simulatedSedEdit` always, and omits `run_in_background` when `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` env var is set.

```typescript
{
  command: string
    // "The command to execute"

  timeout?: number
    // `Optional timeout in milliseconds (max ${Tb1()})`
    // Tb1() = BASH_MAX_TIMEOUT_MS env var or max(600000, defaultTimeout)
    // Default max: 600,000 ms (10 minutes)

  description?: string
    // `Clear, concise description of what this command does...
    //  For simple commands keep it brief (5-10 words):
    //    - ls → "List files in current directory"
    //  For harder-to-parse commands, add enough context.
    //  Never use words like "complex" or "risk".`

  run_in_background?: boolean
    // "Set to true to run this command in the background.
    //  Use TaskOutput to read the output later."
    // Omitted from public schema when CLAUDE_CODE_DISABLE_BACKGROUND_TASKS is set

  dangerouslyDisableSandbox?: boolean
    // "Set this to true to dangerously override sandbox mode
    //  and run commands without sandboxing."

  // --- Internal only (always omitted from public schema): ---
  _simulatedSedEdit?: {
    filePath: string
    newContent: string
  }
    // "Internal: pre-computed sed edit result from preview"
}
```

#### Output Schema (`zSz` — `I.object`)

```typescript
{
  stdout: string
    // "The standard output of the command"

  stderr: string
    // "The standard error output of the command"

  rawOutputPath?: string
    // "Path to raw output file for large MCP tool outputs"

  interrupted: boolean
    // "Whether the command was interrupted"

  isImage?: boolean
    // "Flag to indicate if stdout contains image data"

  backgroundTaskId?: string
    // "ID of the background task if command is running in background"

  backgroundedByUser?: boolean
    // "True if the user manually backgrounded the command with Ctrl+B"

  dangerouslyDisableSandbox?: boolean
    // "Flag to indicate if sandbox mode was overridden"

  returnCodeInterpretation?: string
    // "Semantic interpretation for non-error exit codes with special meaning"

  noOutputExpected?: boolean
    // "Whether the command is expected to produce no output on success"

  structuredContent?: any[]
    // "Structured content blocks"

  persistedOutputPath?: string
    // "Path to the persisted full output in tool-results dir
    //  (set when output is too large for inline)"

  persistedOutputSize?: number
    // "Total size of the output in bytes
    //  (set when output is too large for inline)"

  tokenSaverOutput?: string
    // "Compressed output sent to model when token-saver is active
    //  (UI still uses stdout)"
}
```

#### Constants & Limits

| Constant | Symbol | Value | cli.js Line (approx.) |
|----------|--------|-------|------------------------|
| Default timeout | `Gb1()` / `fb1()` | `BASH_DEFAULT_TIMEOUT_MS` env or **120,000 ms (2 min)** | ~529583 |
| Max timeout | `_Cq()` / `Tb1()` | `BASH_MAX_TIMEOUT_MS` env or **`max(600000, defaultTimeout)`** = **600,000 ms (10 min)** | ~529589 |
| Background offer threshold | `MCq` | **2,000 ms** — after this wall time, UI offers to background | ~529478 |
| Max persisted output file | — | **67,108,864 bytes (64 MB)** | ~529848 |
| maxResultSizeChars | — | **30,000 chars** | ~529720 |

#### Command Classification Sets

Used for `isReadOnly()` determination and display hints:

```javascript
sRz  // search commands: {find, grep, rg, ag, ack, locate, which, whereis}
tRz  // read commands:   {cat, head, tail, less, more, wc, stat, file, strings,
     //                   ls, tree, du, jq, awk, cut, sort, uniq, tr}
WCq  // output-only:    {echo, printf, true, false, :}
eRz  // mutating:       {mv, cp, rm, mkdir, rmdir, chmod, chown, chgrp, touch, ln,
     //                   cd, export, unset, wait}
YSz  // long-running candidates: {npm, yarn, pnpm, node, python, python3, go,
     //                            cargo, make, docker, terraform, webpack, vite,
     //                            jest, pytest, curl, wget, build, test, serve,
     //                            watch, dev}
KSz  // always-background candidate: ["sleep"]
```

#### Permission Model

- **`isEnabled()`:** Always `true` — enabled in all modes.
- **`isConcurrencySafe(A)`:** Delegates to `this.isReadOnly(A)` — safe only when the command is classified as read-only.
- **`isReadOnly(A)`:** Calls `lr6(A.command)` to parse, then `uL1(A, q)` to classify. Returns `allow` (read-only) only when:
  - Command parses successfully
  - `mC(K)` confirms it's a read-only command type (from `sRz` + `tRz` + `WCq` sets)
  - No Windows UNC path (WebDAV vulnerability check via `S26`)
  - No compound `cd` + git (security check)
  - No bare repo + git combination
  - No git outside original CWD when sandbox is enabled
  - All sub-commands pass `LeY()` read-only test
  - Otherwise returns `passthrough` (requires user permission)
- **`checkPermissions(A, q)`:** Calls `$t8(A, q)` — runs the full permission pipeline (rule-based allow/deny/ask).

#### Sandbox Behavior

```javascript
function xr(A) {
  if (!mA.isSandboxingEnabled()) return false;
  if (A.dangerouslyDisableSandbox && mA.areUnsandboxedCommandsAllowed()) return false;
  if (!A.command) return false;
  if (lSz(A.command)) return false;  // sandbox exclusion list check
  return true;
}
```

When sandboxed: `userFacingName()` returns `"SandboxedBash"` if `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` env var is set.

#### Implementation Logic (step-by-step)

1. **`_simulatedSedEdit` shortcut:** If input has `_simulatedSedEdit`, calls `OSz()` directly (returns pre-computed sed edit result); skips all shell execution.

2. **Shell setup:** Calls `LW1(command, signal, "bash", timeout, callback, preventCwdChanges, isSandboxed, shouldBackground)`. Working directory persists between calls; shell state does not. Environment initialized from user's bash/zsh profile.

3. **Explicit background (`run_in_background=true`):** Immediately spawns background task via `_v6.spawn()`, emits `tengu_bash_command_explicitly_backgrounded` analytics, returns `{ stdout:"", stderr:"", code:0, interrupted:false, backgroundTaskId }`.

4. **Fast-path race (first `MCq` = 2,000 ms):** Races command promise against a 2-second timer:
   - Command finishes within 2s → clean up and return immediately.
   - Auto-background triggered within 2s → return background task ID.

5. **Polling loop (slow commands > 2s):** Enters yield loop. Each iteration yields progress: `{ type:"progress", fullOutput, output, elapsedTimeSeconds, totalLines, totalBytes, taskId, timeoutMs? }`. After 2 seconds, UI begins offering background option (via JSX component `sL1`). User pressing Ctrl+B sets `backgroundedByUser: true`.

6. **Timeout background:** If command has `onTimeout` and auto-background is enabled, timeout fires `tengu_bash_command_timeout_backgrounded` event.

7. **Result processing:**
   - Calls `_Sz(command, exitCode, stdout)` for cd-tracking / CWD state update
   - Checks `HCq(command, exitCode, stdout, "")` for `returnCodeInterpretation`
   - Detects `.git/index.lock` in output → emits `tengu_git_index_lock_error`
   - If error and not user-interrupted: constructs `kI` error with stderr, exit code, interrupted flag and **throws**

8. **Large output handling:** If output > 67 MB, truncates to 64 MB, moves to persisted results dir, sets `persistedOutputPath` + `persistedOutputSize`.

9. **Image detection:** `zw4(stdout)` + `ek8()` checks if stdout is base64 image data. If so, sets `isImage: true` and re-encodes via `YW1()`.

10. **Analytics:** Emits `tengu_bash_tool_command_executed` with `{ command_type, stdout_length, stderr_length, exit_code, interrupted }`. Also emits `tengu_code_indexing_tool_used` if command is a known code indexing tool.

11. **Result format for model:** Merges stdout + stderr + backgroundTaskId message. If interrupted: appends `"<error>Command was aborted before completion</error>"`. Strips leading blank lines, trims trailing whitespace from stdout. If `tokenSaverOutput` present, sends that to model instead of stdout (UI still gets raw stdout).

#### Error Handling

| Condition | Behavior |
|-----------|----------|
| Invalid/unparseable commands | `isReadOnly` returns `passthrough` (not an error) |
| Pre-spawn failure (`preSpawnError`) | `throw Error(X.preSpawnError)` |
| Non-zero exit + not user-interrupted | Throws `kI("", stderr, exitCode, interrupted)` |
| User interrupt (Ctrl+C) | `interrupted: true` in result; error suppressed |
| Abort signal | Command process receives abort signal from `abortController` |

#### Prompt Instructions (`OCq()`)

Key behavioral rules injected into the system prompt for the model:
- Working directory persists between commands; shell state does not
- Avoid `find`/`grep`/`cat`/`head`/`tail`/`sed`/`awk`/`echo` — use dedicated tools (Read, Grep, Glob) instead
- Prefer running independent commands in parallel (multiple tool calls in one message)
- Chain dependent commands with `&&`; use `;` only when failure of the prior command is acceptable
- Do not use newlines to separate commands
- Default timeout: `fb1()` ms (120s); configurable up to `Tb1()` ms (600s)
- Git behavioral rules: no `--no-verify`, no skipping hooks, prefer new commits over amends
- Sleep guidelines: don't sleep between immediately-runnable commands; use `run_in_background` for long-running commands

---

*End of Part 1. Continue in Part 2 for Web Tools, Agent & Task Tools, and User Interaction & Planning tools.*
## 5. Web Tools

Web tools provide HTTP fetch and search capabilities. Both declare `shouldDefer: true` and `maxResultSizeChars: 100,000`. Both are read-only and concurrency-safe.

---

### WebFetch

**Tool name:** `"WebFetch"` · **User-facing name:** `"Fetch"`  
**Variable:** `UP` (tool object), `VM` (name string)  
**Source:** cli.js ~line 238 region  
**searchHint:** `"fetch and extract content from a URL"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`

#### Input Schema (`K9z`)

```typescript
I.strictObject({
  url: string (URL-validated)
    // "The URL to fetch content from"

  prompt: string
    // "The prompt to run on the fetched content"
})
```

#### Output Schema (`Y9z`)

```typescript
I.object({
  bytes: number
    // "Size of the fetched content in bytes"

  code: number
    // "HTTP response code"

  codeText: string
    // "HTTP response code text"

  result: string
    // "Processed result from applying the prompt to the content"

  durationMs: number
    // "Time taken to fetch and process the content"

  url: string
    // "The URL that was fetched"
})
```

#### Constants and Limits

| Symbol | Value | Meaning |
|--------|-------|---------|
| `n5z` | 900,000 ms (15 min) | Cache TTL |
| `r5z` | 52,428,800 bytes (50 MB) | Cache max total size |
| `a5z` | 2,000 chars | Max URL length (`U2q` validator) |
| `s5z` | 10,485,760 bytes (10 MB) | `maxContentLength` for axios |
| `t5z` | 60,000 ms (60 s) | Axios request timeout |
| `e5z` | 10,000 ms (10 s) | Preflight domain check timeout |
| `oo6` | 100,000 chars | Truncation threshold before passing content to AI model (`bc8`) |

**Cache object (`Sc8`):** LRU cache via `ck` (lru-cache):
```javascript
new ck({
  maxSize: 52428800,        // 50 MB total
  sizeCalculation: (A) => Math.max(1, Buffer.byteLength(A.content)),
  ttl: 900000,              // 15 minutes
})
```

#### Permission Model

- **`isEnabled()`:** Always `true`.
- **`isReadOnly()`:** Always `true`.
- **`isConcurrencySafe()`:** Always `true`.
- **`checkPermissions(A, q)`:**
  1. Checks `kR1` preapproved domains — if hostname matches exactly or hostname+path-prefix matches, returns `{behavior: "allow", decisionReason: {type: "other", reason: "Preapproved host"}}`.
  2. Computes domain-based permission key via `z9z(A)`.
  3. Checks existing "deny" rules → `{behavior: "deny"}`.
  4. Checks existing "ask" rules → `{behavior: "ask"}`.
  5. Checks existing "allow" rules → `{behavior: "allow"}`.
  6. Default: `{behavior: "ask"}` — user must explicitly allow.

**Preapproved domains (`kR1`):** 80+ entries including `platform.claude.com`, `code.claude.com`, `modelcontextprotocol.io`, `github.com/anthropics`, `docs.python.org`, `developer.mozilla.org`, `react.dev`, `nextjs.org`, `nodejs.org`, `www.typescriptlang.org`, `tailwindcss.com`, `docs.aws.amazon.com`, `cloud.google.com`, `kubernetes.io`, `www.docker.com`, `vercel.com/docs`, `docs.netlify.com`, `git-scm.com`, and many other language/framework documentation sites.

#### Implementation Logic

**`validateInput(A)`:**
- Attempts `new URL(q)` — if throws: `{result: false, message: 'Error: Invalid URL "...". The URL provided could not be parsed.', meta: {reason: "invalid_url"}, errorCode: 1}`.
- If valid: `{result: true}`.

**`call({ url, prompt }, context)`:**

1. **Record start time** via `Date.now()`.

2. **Fetch via `Ic8(url, abortController)`:**
   - **URL validation (`U2q`):** URL must be ≤ 2,000 chars, parseable, no username/password, hostname must have ≥ 2 dot-segments.
   - **Cache check:** If `Sc8` has an entry for URL, return cached `{bytes, code, codeText, content, contentType, persistedPath, persistedSize}`.
   - **HTTP→HTTPS upgrade:** `http:` URLs are silently upgraded to `https:`.
   - **Preflight domain check (`A9z(hostname)`):** Unless `skipWebFetchPreflight` config is set, calls `https://api.anthropic.com/api/web/domain_info?domain=<encoded>` with 10 s timeout:
     - `can_fetch === true` → `{status: "allowed"}`
     - `can_fetch === false` → `{status: "blocked"}` → throws `DomainBlockedError`
     - Network failure → `{status: "check_failed"}` → throws `DomainCheckFailedError`
   - **Fetch via axios (`hc8`):** `timeout: 60,000 ms`, `maxRedirects: 0` (manual), `responseType: "arraybuffer"`, `maxContentLength: 10 MB`, `Accept: "text/markdown, text/html, */*"`.
   - **Redirect handling:** 301/302/307/308 — same-domain redirects (matching protocol + port + hostname, stripping `www.`) auto-follow recursively. Cross-domain redirects return `{type: "redirect", originalUrl, redirectUrl, statusCode}`.
   - **Egress proxy block:** 403 + `x-proxy-error: blocked-by-allowlist` → throws `EgressBlockedError` with JSON `{error_type: "EGRESS_BLOCKED", domain, message}`.
   - **Binary content:** If content-type is binary (`fYq($)`), persists to temp file, records `persistedPath` + `persistedSize`.
   - **HTML conversion:** If `content-type` includes `text/html`, converts to markdown via `turndown` library.
   - **Cache store:** Stores result in `Sc8` keyed by original URL.

3. **Redirect result path:** Returns redirect message to model with instructions to re-call using the redirect URL.

4. **Content processing (`bc8(prompt, content, signal, isNonInteractive, isPreapproved)`):**
   - If URL is in `kR1` (preapproved) AND content-type is `text/markdown` AND content < `oo6` (100,000 chars) → return content directly without AI processing.
   - Otherwise → calls `PG()` (a small/fast Claude model via `querySource: "web_fetch_apply"`) with system prompt + content + user prompt. Content is truncated at `oo6` chars with `[Content truncated due to length...]` appended before AI processing.

5. **Binary content note:** If binary file was persisted, appends `[Binary content (<type>, <size>) also saved to <path>]` to result.

6. **Return:** `{data: {bytes, code, codeText, result, durationMs, url}}`.

**`mapToolResultToToolResultBlockParam`:** Returns `{tool_use_id, type: "tool_result", content: result}` — just the `result` string.

#### Error Classes

| Class | `name` property | Message |
|-------|-----------------|---------|
| `yc8` | `"DomainBlockedError"` | `"Claude Code is unable to fetch from <domain>"` |
| `Rc8` | `"DomainCheckFailedError"` | `"Unable to verify if domain <X> is safe to fetch. This may be due to network restrictions or enterprise security policies blocking claude.ai."` |
| `Q2q` | `"EgressBlockedError"` | JSON string: `{error_type: "EGRESS_BLOCKED", domain, message: "Access to <domain> is blocked by the network egress proxy."}` |

#### Prompt (`V78`)

Key behavioral rules injected into the system prompt:
- If an MCP web fetch tool is available, prefer it (fewer restrictions).
- HTTP URLs are auto-upgraded to HTTPS.
- Responses are cached for 15 minutes.
- Cross-domain redirects require re-calling with the redirect URL.
- GitHub URLs: prefer `gh` CLI via Bash instead of WebFetch.
- Results may be summarized if content is very large.
- Tool is read-only; does not modify files.
- **Special MCP note:** If `OW` (MCP integration tool) is among session tools, the prompt adds: `"IMPORTANT: WebFetch WILL FAIL for authenticated or private URLs... use ${OW} first to find a specialized tool."`

#### Conditional Features

- `isEnabled()` always returns `true`.
- `skipWebFetchPreflight` config flag skips the Anthropic domain safety check API call.
- Preapproved domains (`kR1`): auto-allowed without user prompt AND skip AI processing for markdown content under 100,000 chars.

---

### WebSearch

**Tool name:** `"WebSearch"`  
**Variable:** `PS1` (tool object), `tV` (name string)  
**Source:** cli.js ~line 421 region  
**searchHint:** `"search the web for current information"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`

#### Input Schema (`X9z`)

```typescript
I.strictObject({
  query: string (min length: 2)
    // "The search query to use"

  allowed_domains?: string[]
    // "Only include search results from these domains"

  blocked_domains?: string[]
    // "Never include search results from these domains"
})
```

Validation rejects requests specifying both `allowed_domains` and `blocked_domains` simultaneously.

#### Output Schema (`W9z`)

```typescript
// Internal result item (P9z):
I.object({
  tool_use_id: string,
  content: Array<{
    title: string,   // "The title of the search result"
    url: string,     // "The URL of the search result"
  }>,
})

// Full output (W9z):
I.object({
  query: string
    // "The search query that was executed"

  results: Array<P9z | string>
    // "Search results and/or text commentary from the model"

  durationSeconds: number
    // "Time taken to complete the search operation"
})
```

#### Constants and Limits

| Constant | Value | Meaning |
|----------|-------|---------|
| `max_uses` in `Z9z` | 8 | Max web search uses per assistant turn |
| `query.min(2)` | 2 chars | Minimum query length |

**Server-side tool descriptor (`Z9z(A)`):**
```javascript
{
  type: "web_search_20250305",
  name: "web_search",
  allowed_domains: A.allowed_domains,
  blocked_domains: A.blocked_domains,
  max_uses: 8,
}
```
WebSearch is a **server-side tool** — the prefix `web_search_20250305` signals that searching is performed by Anthropic's API infrastructure, not client-side code.

#### Permission Model

- **`isEnabled()`:**
  ```javascript
  let api = D7()   // API mode
  let model = d5() // current model ID
  if (api === "firstParty") return true;
  if (api === "vertex")
    return model.includes("claude-opus-4") ||
           model.includes("claude-sonnet-4") ||
           model.includes("claude-haiku-4");
  if (api === "foundry") return true;
  return false;
  ```
  - **firstParty mode:** Always enabled.
  - **Vertex AI:** Only on claude-opus-4, claude-sonnet-4, claude-haiku-4 model families.
  - **AWS Bedrock/foundry:** Enabled.
  - **Other modes (e.g., third-party):** Disabled.

- **`isReadOnly()`:** Always `true`.
- **`isConcurrencySafe()`:** Always `true`.
- **`checkPermissions(A)`:** Returns `{behavior: "passthrough", message: "WebSearchTool requires permission.", suggestions: [{type: "addRules", rules: [{toolName: "WebSearch"}], behavior: "allow", destination: "localSettings"}]}`.

#### Implementation Logic

**`validateInput(A)`:**
- Empty query → `{result: false, message: "Error: Missing query", errorCode: 1}`.
- Both `allowed_domains` and `blocked_domains` specified → `{result: false, message: "Error: Cannot specify both allowed_domains and blocked_domains in the same request", errorCode: 2}`.
- Otherwise → `{result: true}`.

**`call(A, q, K, Y, z)`** (`z` = optional `onProgress` callback):

1. **Record start time** via `performance.now()`.
2. **Build user message:** `A8({content: "Perform a web search for the query: " + query})`.
3. **Build server tool descriptor:** `Z9z(A)` = `{type: "web_search_20250305", name: "web_search", allowed_domains, blocked_domains, max_uses: 8}`.
4. **Feature flag `tengu_plum_vx3`:** If enabled, forces a specific smaller model (`Fj()`) with thinking disabled and `toolChoice: {type: "tool", name: "web_search"}`.
5. **Stream API call (`ST6(...)`):**
   - System prompt: `"You are an assistant for performing a web search tool use"`.
   - `extraToolSchemas` injects the Anthropic server-side web search tool descriptor.
   - Returns a streaming async iterator.
6. **Stream processing loop:** For each streamed event:
   - `content_block_start` with `server_tool_use` → captures tool use ID, resets accumulator.
   - `content_block_delta` with `input_json_delta` → accumulates JSON; regex-extracts `query` field from partial JSON → calls `z({type: "query_update", query})`.
   - `content_block_start` with `web_search_tool_result` → calls `z({type: "search_results_received", resultCount, query})`.
7. **Result assembly:** Filters assistant messages, flattens content arrays, calls `G9z(contentBlocks, query, durationSeconds)`. Returns `{data: {query, results, durationSeconds}}`.

**`mapToolResultToToolResultBlockParam`:** Formats results as plaintext:
```
Web search results for query: "<query>"

[For each result:
  - If string: raw string + newline
  - If object with content: "Links: <formatted URLs>\n"
  - If object with empty content: "No links found.\n"
]

REMINDER: You MUST include the sources above in your response to the user using markdown hyperlinks.
```

#### Prompt (`ftA()`)

- Allows Claude to search the web and use results to inform responses.
- Provides up-to-date information for current events and recent data.
- Returns results as search result blocks with markdown hyperlink formatting.
- **`CRITICAL REQUIREMENT`:** After answering, Claude MUST include a `Sources:` section listing all relevant URLs as markdown hyperlinks. This is described as MANDATORY.
- Domain filtering supported via `allowed_domains`/`blocked_domains`.
- `"Web search is only available in the US"` — hardcoded in prompt.
- Current month is dynamically injected via `GtA()` (e.g., `"March 2026"`) with instruction to use the correct year in queries.

#### Conditional Features

- `tengu_plum_vx3` feature flag: Forces a smaller model with forced tool choice, disables thinking.
- Vertex AI: Limited to claude-opus-4/sonnet-4/haiku-4 model families only.
- Geographic restriction: US-only (enforced server-side; mentioned in prompt).
- `max_uses: 8` limits total search invocations per turn at the API level.

---

## 6. Agent & Task Tools

Seven tools manage agent spawning and task lifecycle. The task management tools (TaskCreate, TaskGet, TaskUpdate, TaskList) are gated by `iH()` and disabled in non-interactive API sessions by default. TaskOutput and TaskStop are always enabled.

### Tool Aliases

```javascript
aUA = {
  Task: "Agent",            // "Task" is an alias for the Agent tool
  KillShell: "TaskStop",    // legacy alias
  AgentOutputTool: "TaskOutput",
  BashOutputTool: "TaskOutput",
}
```

### `iH()` — Tasks Feature Gate (line 231665)

```javascript
function iH() {
  if ($1(process.env.CLAUDE_CODE_ENABLE_TASKS)) return true;
  return !u7();  // u7() = !isInteractive; so iH() = isInteractive
}
```
Tasks tools are enabled in interactive sessions OR when `CLAUDE_CODE_ENABLE_TASKS` env var is set. Disabled in non-interactive/API sessions by default.

### Task State Machine

```
pending → in_progress → completed
                     ↘ deleted  (hard removal, TaskUpdate only)
```

Status enum (`kY6()`): `"pending" | "in_progress" | "completed"` (TaskUpdate additionally accepts `"deleted"`).

---

### Agent

**Tool name:** `"Agent"` · **Alias:** `"Task"`  
**Constant:** `var Tq = "Agent"` (line 64870), `XK6 = "Task"` (line 64871)  
**searchHint:** `"delegate work to a subagent"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** not set (runs synchronously in `call()`)

#### Input Schema

Built via `wQ8()` → `L8z()` → `E8z()` composition.

**Base schema `E8z()` (line 418877):**
```typescript
{
  description: string     // required — "A short (3-5 word) description of the task"
  prompt: string          // required — "The task for the agent to perform"
  subagent_type?: string  // "The type of specialized agent to use for this task"
  resume?: string         // "Optional agent ID to resume from. If provided, the agent will continue from the previous execution transcript."
  run_in_background?: boolean  // "Set to true to run this agent in the background. You will be notified when it completes."
}
```

**`L8z()` extends `E8z()` with (line 418898):**
```typescript
{
  // ...all E8z fields
  name?: string           // "Name for the spawned agent"
  team_name?: string      // "Team name for spawning. Uses current team context if omitted."
  mode?: cUA()            // Permission mode enum — e.g., "plan" to require plan approval
  isolation?: "worktree"  // 'Isolation mode. "worktree" creates a temporary git worktree'
  cwd?: string            // "Absolute path to run the agent in. Mutually exclusive with isolation: 'worktree'."
}
```

**Model-facing schema `wQ8()` (line 418927):**
```typescript
// L8z() minus cwd always; minus run_in_background when
// CLAUDE_CODE_DISABLE_BACKGROUND_TASKS is set OR in API mode (UT6())
let A = L8z().omit({ cwd: true });
return ay1 || UT6() ? A.omit({ run_in_background: true }) : A;
```
`cwd` is **always** omitted from the model-facing schema.

#### Output Schema (`R8z()`, line 418957)

Union of two shapes:

**Shape A — synchronous completed agent:**
```typescript
{
  status: "completed"
  agentId: string
  content: Array<{ type: "text", text: string }>
  totalToolUseCount: number
  totalDurationMs: number
  totalTokens: number
  usage: {
    input_tokens: number
    output_tokens: number
    cache_creation_input_tokens: number | null
    cache_read_input_tokens: number | null
    server_tool_use: { web_search_requests: number, web_fetch_requests: number } | null
    service_tier: "standard" | "priority" | "batch" | null
    cache_creation: { ephemeral_1h_input_tokens: number, ephemeral_5m_input_tokens: number } | null
  }
  prompt: string
}
```

**Shape B — background-launched agent:**
```typescript
{
  status: "async_launched"
  agentId: string          // "The ID of the async agent"
  description: string      // "The description of the task"
  prompt: string           // "The prompt for the agent"
  outputFile: string       // "Path to the output file for checking agent progress"
  canReadOutputFile?: boolean  // "Whether the calling agent has Read/Bash tools to check progress"
}
```

A third status `"teammate_spawned"` (line 419057) is returned when a named teammate is spawned via the Agent Teams feature.

#### Permission Model

- **`checkPermissions`:** Always `{behavior: "allow"}` — no restrictions at the permission layer.
- Parallelism is user-instructed; no concurrency limit found in schema or call logic.

#### Implementation Logic (`call()`, line 418996+)

Parameters: `prompt`, `subagent_type`, `description`, `resume`, `run_in_background`, `name`, `team_name`, `mode`, `isolation`, `cwd`.

1. **Team/plan-mode checks:**
   - If `team_name` provided but Agent Teams disabled (`!Z7()`): throw `"Agent Teams is not yet available on your plan."`
   - If current agent is a teammate (`Oz()`) and both `team_name` and `name` are provided: throw `"Teammates cannot spawn other teammates — the team roster is flat. To spawn a subagent instead, omit the name parameter."`
   - If in-process teammate (`AW()`) and `team_name` and `run_in_background === true`: throw `"In-process teammates cannot spawn background agents. Use run_in_background=false for synchronous subagents."`

2. **Named teammate spawn** (if `team_name` resolved and `name` provided): Looks up agent definition by `subagent_type`, calls `HKq(...)`, returns `{status: "teammate_spawned", ...}`.

3. **Resume path** (if `resume` agentId provided): Validates task is not running (throws if `status === "running"`). Loads previous transcript via `Ov6(gW(resume))`. Throws `"No transcript found for agent ID: {id}"` if not found.

4. **Agent definition resolution:** Resolves agent type from `subagent_type` or defaults. Sets `isAsync` flag: `z === true || agentDef.background === true`.

5. **Isolation (worktree):** If `isolation === "worktree"`, creates a git worktree via `Oo6()`.

6. **Async/sync decision:**
   ```javascript
   let isActuallyAsync =
     (z === true || agentDef.background === true || isInProactiveMode) &&
     !ay1 &&  // CLAUDE_CODE_DISABLE_BACKGROUND_TASKS not set
     !d;
   ```
   - Async: returns `{status: "async_launched", agentId, description, prompt, outputFile, canReadOutputFile}`.
   - Sync: awaits completion, returns `{status: "completed", ...metrics}`.

7. **Permission context:** Uses `agentDef.permissionMode ?? "acceptEdits"` for the spawned agent's permission mode.

#### Constants and Limits

- `maxResultSizeChars: 1e5` (100,000 chars)
- `ay1 = $1(process.env.CLAUDE_CODE_DISABLE_BACKGROUND_TASKS)` — disables background tasks entirely when set
- Background tasks also disabled in API mode (`UT6()`)
- `run_in_background` removed from schema when background is disabled

#### Agent Teams Feature Gate (`Z7()`, line 72146)

```javascript
function Z7() {
  if (!$1(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS) && !dK3()) return false;
  if (!p8("tengu_amber_flint", true)) return false;
  return true;
}
// dK3() = process.argv.includes("--agent-teams")
// p8() = feature flag check
```

---

### TaskCreate

**Tool name:** `"TaskCreate"`  
**Constant:** `var US = "TaskCreate"` (line 250368)  
**searchHint:** `"create a task in the task list"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`  
**isEnabled:** `() => iH()` (interactive session or `CLAUDE_CODE_ENABLE_TASKS`)  
**isConcurrencySafe:** `() => true`  
**isReadOnly:** `() => false`

#### Input Schema (`kYz()`, line 448826)

```typescript
I.strictObject({
  subject: string
    // required — "A brief title for the task"

  description: string
    // required — "A detailed description of what needs to be done"

  activeForm?: string
    // 'Present continuous form shown in spinner when in_progress (e.g., "Running tests")'

  metadata?: Record<string, unknown>
    // "Arbitrary metadata to attach to the task"
})
```

#### Output Schema (`EYz()`, line 448842)

```typescript
I.object({
  task: I.object({ id: string, subject: string })
})
```

#### Implementation Logic

1. Calls `j01(GT(), { subject, description, activeForm, status: "pending", owner: undefined, blocks: [], blockedBy: [], metadata })`.
2. Opens task panel UI (`expandedView: "tasks"`) if not already open.
3. Returns `{data: {task: {id: newId, subject}}}`.
4. **Tool result message:** `"Task #${id} created successfully: ${subject}"`.

#### Permission Model

`checkPermissions`: always `{behavior: "allow"}` — no user confirmation needed.

---

### TaskGet

**Tool name:** `"TaskGet"`  
**Constant:** `var c16 = "TaskGet"` (line 289965)  
**searchHint:** `"retrieve a task by ID"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`  
**isEnabled:** `() => iH()`  
**isConcurrencySafe:** `() => true`  
**isReadOnly:** `() => true`

#### Input Schema (`LYz()`, line ~448958)

```typescript
I.strictObject({
  taskId: string  // "The ID of the task to retrieve"
})
```

#### Output Schema (`yYz()`, line ~448963)

```typescript
I.object({
  task: I.object({
    id: string
    subject: string
    description: string
    status: "pending" | "in_progress" | "completed"
    blocks: string[]    // task IDs this task blocks
    blockedBy: string[] // task IDs blocking this task
  }).nullable()         // null if task not found
})
```

**Note:** The full task store object (line 231938+) also includes `activeForm?`, `owner?`, and `metadata?`, but TaskGet's output schema only exposes the six fields above.

#### Implementation Logic

Calls `jF(GT(), taskId)` — reads task from store by ID. Returns `{data: {task: taskData | null}}`.

#### Permission Model

`checkPermissions`: always `{behavior: "allow"}`.

---

### TaskUpdate

**Tool name:** `"TaskUpdate"`  
**Constant:** `var KL = "TaskUpdate"` (line 250369)  
**searchHint:** `"update a task"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`  
**isEnabled:** `() => iH()`  
**isConcurrencySafe:** `() => true` (uses file locking)  
**isReadOnly:** `() => false`

#### Input Schema (`RYz()`, line 449140+)

```typescript
I.strictObject({
  taskId: string
    // required — "The ID of the task to update"

  subject?: string
    // "New subject for the task"

  description?: string
    // "New description for the task"

  activeForm?: string
    // 'Present continuous form shown in spinner when in_progress'

  status?: "pending" | "in_progress" | "completed" | "deleted"
    // "New status for the task" — "deleted" causes hard removal

  addBlocks?: string[]
    // "Task IDs that this task blocks"

  addBlockedBy?: string[]
    // "Task IDs that block this task"

  owner?: string
    // "New owner for the task"

  metadata?: Record<string, unknown>
    // "Metadata keys to merge into the task. Set a key to null to delete it."
})
```

#### Output Schema (`SYz()`, line 449168+)

```typescript
I.object({
  success: boolean
  taskId: string
  updatedFields: string[]              // list of field names that were changed
  error?: string
  statusChange?: { from: string, to: string }
})
```

#### Implementation Logic

1. Opens task panel UI if not already open.
2. Looks up task by `taskId` via `jF(GT(), taskId)` — if not found, returns `{success: false, error: "Task not found"}`.
3. Builds diff object `D` and tracks changed fields in `M`; only updates fields that differ from current values.
4. **Auto-owner assignment:** If Agent Teams feature (`Z7()`) is active AND `status === "in_progress"` AND no `owner` specified AND task has no current owner, automatically sets `owner = V9()` (current agent name).
5. **Metadata merge:** Merges keys into existing metadata; keys set to `null` are deleted.
6. **Blocks/BlockedBy:** Calls `SN8(j, blockerId, blockedId)` to add dependency edges.
7. Returns `{data: {success: true, taskId, updatedFields, statusChange}}`.

#### Permission Model

`checkPermissions`: always `{behavior: "allow"}`.

---

### TaskList

**Tool name:** `"TaskList"`  
**Constant:** `var l16 = "TaskList"` (line 289966)  
**searchHint:** `"list all tasks"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`  
**isEnabled:** `() => iH()`  
**isConcurrencySafe:** `() => true`  
**isReadOnly:** `() => true`

#### Input Schema (`CYz()`, line 449457)

```typescript
I.strictObject({})  // no parameters
```

#### Output Schema (`hYz()`, line 449458+)

```typescript
I.object({
  tasks: Array<{
    id: string
    subject: string
    status: "pending" | "in_progress" | "completed"
    owner?: string
    blockedBy: string[]  // filtered to only incomplete (non-completed) blockers
  }>
})
```

#### Implementation Logic

1. `GT()` — gets current session context.
2. `DP(A)` — loads all tasks for the session.
3. Filters out internal tasks: `.filter((z) => !z.metadata?._internal)` — tasks with `metadata._internal === true` are system-internal and never shown to agents.
4. Builds a `completedSet` of task IDs with `status === "completed"`.
5. For each task returns: `{id, subject, status, owner, blockedBy: blockedBy.filter(id => !completedSet.has(id))}`.
   - **Key behavior:** `blockedBy` is pruned to only show IDs of tasks that have NOT yet completed, so agents only see active blockers.

#### Permission Model

`checkPermissions`: always `{behavior: "allow"}`.

---

### TaskOutput

**Tool name:** `"TaskOutput"`  
**Constant:** `var SI = "TaskOutput"` (line 64880)  
**Aliases:** `AgentOutputTool`, `BashOutputTool`  
**User-facing name:** `"Task Output"`  
**searchHint:** `"read output/logs from a background task"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`  
**isEnabled:** `() => true` (always — unlike task management tools)  
**isConcurrencySafe:** `() => true`  
**isReadOnly:** `() => true`

#### Input Schema (`j9z()`, line 445009)

```typescript
I.strictObject({
  task_id: string
    // required — "The task ID to get output from"

  block: boolean
    // default: true — "Whether to wait for completion"

  timeout: number
    // min: 0, max: 600000, default: 30000 — "Max wait time in ms"
})
```

#### Validation

- If `task_id` is empty: `{result: false, message: "Task ID is required", errorCode: 1}`
- If task not in `appState.tasks`: `{result: false, message: "No task found with ID: {id}", errorCode: 2}`

#### Implementation Logic (`call()`, line 445074+)

**Non-blocking mode (`block === false`):**
- If task is NOT running/pending (already done): marks task notified, returns `{retrieval_status: "success", task: DS1(H)}`.
- If still running/pending: returns `{retrieval_status: "not_ready", task: DS1(H)}`.

**Blocking mode (`block === true`, default):**
- Emits progress event: `{type: "waiting_for_task", taskDescription, taskType}`.
- Calls `J9z(task_id, getAppState, timeout, abortController)` — polling loop:
  - Polls every **100 ms**.
  - Aborts if signal is aborted.
  - Returns task when `status !== "running" && status !== "pending"`.
  - Returns `null` on timeout.
- If `null` returned: `{retrieval_status: "timeout", task: null}`.
- If still running after timeout: `{retrieval_status: "timeout", task: DS1(task)}`.
- On success: `{retrieval_status: "success", task: DS1(H)}`.

#### Constants

| Symbol | Value | Meaning |
|--------|-------|---------|
| Default `timeout` | 30,000 ms | Default wait time |
| Max `timeout` | 600,000 ms (10 min) | Schema-enforced maximum |
| `Qc8` | 32,000 chars | Default output truncation limit |
| `Fc8` | 160,000 chars | Max output (via `TASK_MAX_OUTPUT_LENGTH` env var) |

Truncation format: `"[Truncated. Full output: {path}]\n"` + last N chars of output.

Task handler types registered with `w9z()` (line 444401): `[_v6, lL1, cKq]` — corresponding to local bash shell, async agent, and remote session handlers.

#### Permission Model

`checkPermissions`: always `{behavior: "allow"}`.

---

### TaskStop

**Tool name:** `"TaskStop"`  
**Constant:** `var RI = "TaskStop"` (line 64873)  
**Alias:** `KillShell`  
**User-facing name:** `() => JH() ? "" : "Stop Task"` (hidden in some contexts)  
**searchHint:** `"kill a running background task"`  
**maxResultSizeChars:** `100,000` (`1e5`)  
**shouldDefer:** `true`  
**isEnabled:** `() => true` (always enabled)  
**isConcurrencySafe:** `() => true`  
**isReadOnly:** `() => false`

#### Input Schema (`$9z()`, line 444502)

```typescript
I.strictObject({
  task_id?: string
    // "The ID of the background task to stop"
    // Preferred field

  shell_id?: string
    // "Deprecated: use task_id instead"
    // Legacy alias; resolved as task_id ?? shell_id
})
// Neither is marked required at schema level;
// validation enforces that at least one must be present
```

#### Output Schema (`O9z()`, line 444512)

```typescript
I.object({
  message: string   // "Status message about the operation"
  task_id: string   // "The ID of the task that was stopped"
  task_type: string // "The type of the task that was stopped"
  command?: string  // "The command or description of the stopped task"
})
```

#### Validation (`validateInput()`)

```javascript
async validateInput({ task_id, shell_id }, { getAppState }) {
  const id = task_id ?? shell_id;
  if (!id)
    return { result: false, message: "Missing required parameter: task_id", errorCode: 1 };
  const task = getAppState().tasks?.[id];
  if (!task)
    return { result: false, message: `No task found with ID: ${id}`, errorCode: 1 };
  if (!HS1(task.type))
    return { result: false, message: `Task ${id} has unsupported type: ${task.type}`, errorCode: 2 };
  if (task.status !== "running")
    return { result: false, message: `Task ${id} is not running (status: ${task.status})`, errorCode: 3 };
  return { result: true };
}
```

#### Implementation Logic

1. Resolves `id = task_id ?? shell_id`.
2. If no id: throws `"Missing required parameter: task_id"`.
3. Calls `JS1(id, { abortController, getAppState, setAppState })` — the actual stop handler.
4. Returns: `{data: {message: "Successfully stopped task: {taskId} ({command})", task_id, task_type, command}}`.

#### Error Codes (Task Tools Summary)

| Tool | Code | Meaning |
|------|------|---------|
| TaskOutput | 1 | Missing `task_id` |
| TaskOutput | 2 | Task not found |
| TaskStop | 1 | Missing `task_id` OR task not found |
| TaskStop | 2 | Unsupported task type |
| TaskStop | 3 | Task not in `"running"` status |

#### Concurrency and File Locking

All Task* tools declare `isConcurrencySafe: () => true`. The task store uses file locking (`Wp6.lock`) with retry config `{retries: 10, minTimeout: 5, maxTimeout: 100}` for concurrent writes.

---

## 7. User Interaction & Planning

Three tools manage user-facing Q&A and plan-mode transitions. All declare `shouldDefer: true`. All three are members of `cT6` — the set of tools hidden from the tool list by default.

---

### AskUserQuestion

**Tool name:** `"AskUserQuestion"`  
**Constant:** `var b_ = "AskUserQuestion"` (line 289923)  
**Description constant:** `Xf4` (line 289926) — `"Asks the user multiple choice questions..."`  
**Max header chars:** `Df4 = 12`  
**shouldDefer:** `true`  
**requiresUserInteraction:** `true`

#### Input Schema

```typescript
I.strictObject({
  questions: I.array(
    I.object({
      question: string
        // Complete question ending with ?

      header: string
        // Max 12 chars — chip label (e.g., "Auth method")

      options: I.array(
        I.object({
          label: string
            // 1-5 words display text

          description: string
            // Explanation of the option

          preview?: string
            // Optional preview content (markdown or HTML fragment)
        })
      ).min(2).max(4)
        // 2-4 options required per question

      multiSelect?: boolean
        // default: false
    })
  ).min(1).max(4)
    // 1-4 questions total

  // Additional fields from C9z extension:
  answers?: Record<string, string>
    // Pre-filled answers (optional)

  annotations?: Record<string, {
    preview?: string,
    notes?: string
  }>

  metadata?: {
    source?: string    // Analytics tracking, e.g. "remember"
  }
})
// Refinement: all question texts must be unique;
// all option labels within each question must be unique
```

#### Output Schema

```typescript
{
  questions: Question[]
  answers: Record<string, string>   // question text → answer; multi-select = comma-separated
  annotations?: Record<string, { preview?: string, notes?: string }>
}
```

#### Preview Feature

Two preview formats depending on `w81()` (preview format setting):
- **`"markdown"`:** ASCII mockups, code snippets, multi-line text in monospace box; side-by-side layout when any option has preview; **only for single-select**.
- **`"html"`:** Self-contained HTML fragment (no `<html>/<body>/<script>/<style>`); inline styles only.

**HTML validation (`u9z`):** Rejects full documents, `<script>/<style>` tags, and non-HTML content when format is `"html"`.

#### Implementation Logic

1. `call({ questions, answers={}, annotations })`: returns `{data: {questions, answers, annotations}}`.
2. `mapToolResultToToolResultBlockParam`: formats as `"question"="answer"` pairs with optional preview/notes.
3. Tool result string: `"User has answered your questions: \"question\"=\"answer\", ..."`

#### Permission Model

- **Auto mode:** Denied — `"Auto mode: Questions interrupt continuous execution. Make reasonable assumptions and proceed."`
- **All other modes:** `{behavior: "ask", message: "Answer questions?", updatedInput: A}`.
- `requiresUserInteraction(): true`.

#### Prompt Notes

The prompt explicitly warns:
- Do NOT use to ask "Is my plan ready?" or "Should I proceed?" — use ExitPlanMode for that.
- Do NOT reference "the plan" in questions (user cannot see the plan until ExitPlanMode is called).

---

### EnterPlanMode

**Tool name:** `"EnterPlanMode"`  
**Constant:** `var p16 = "EnterPlanMode"` (line 289922)  
**shouldDefer:** `true`  
**isReadOnly:** not set (writes state)  
Member of `cT6` (hidden from tool list by default).

#### Input Schema

```typescript
I.strictObject({})  // No parameters
```

#### Output Schema

```typescript
{
  message: string  // "Entered plan mode. You should now focus on exploring..."
}
```

#### Implementation Logic

1. Throws if called in agent context: `if (q.agentId) throw Error("EnterPlanMode tool cannot be used in agent contexts")`.
2. Reads current permission context mode; logs mode transition via `Cp()`.
3. Calls `q.setAppState()` to update `toolPermissionContext` with `{type: "setMode", mode: "plan", destination: "session"}` via `nz(Kk6(ctx), action)`.
4. Returns `{data: {message: "Entered plan mode..."}}`.
5. **Tool result content:** In interview phase (`TO() === true`): adds `"DO NOT write or edit any files except the plan file"`. Otherwise adds 6-step workflow instructions ending with `"Remember: DO NOT write or edit any files yet."`

#### Permission Model

- `checkPermissions`: always `{behavior: "allow"}` — no user confirmation required.
- Cannot be used in agent contexts (runtime throws).

#### When to Use

Prompt varies by permission mode:
- **Default/plan mode (`qYz()`):** Use proactively for non-trivial tasks — new features, multiple approaches, code modifications, architectural decisions, multi-file changes, unclear requirements, when user preferences matter.
- **Auto mode (`k$q("auto")`):** Much more restrictive — only when user explicitly requests planning or there is exceptional architectural ambiguity.

#### Plan Mode Constants

```javascript
BZ1 = { TURNS_SINCE_WRITE: 10, TURNS_BETWEEN_REMINDERS: 10 }
a_4 = { TURNS_BETWEEN_ATTACHMENTS: 5, FULL_REMINDER_EVERY_N_ATTACHMENTS: 5 }
U4Y = { TOKEN_COOLDOWN: 5000 }
p4Y = { TURNS_BETWEEN_REMINDERS: 10 }
```

---

### ExitPlanMode

**Tool name:** `"ExitPlanMode"`  
**Constants:** `var qL = aM = "ExitPlanMode"` (line 248318)  
**shouldDefer:** `true`  
**isReadOnly:** `() => false` (writes plan file, changes permission state)  
Member of `cT6` (hidden from tool list by default).

#### Input Schema (`E_q`)

```typescript
I.strictObject({
  allowedPrompts?: Array<{
    tool: "Bash"
    prompt: string  // Semantic description, e.g. "run tests", "install dependencies"
  }>
  plan?: string    // Injected programmatically from plan file on disk — NOT provided by Claude
  // ...any additional fields via .passthrough()
})
```

**Important:** The `plan` field is injected by `normalizeToolInput` from the plan file path returned by `eD(agentId)`. Claude does NOT pass the plan as a parameter.

#### Output Schema

```typescript
{
  plan: string | null
  isAgent: boolean
  filePath?: string
  hasTaskTool?: boolean
  awaitingLeaderApproval?: boolean
  requestId?: string
  isUltraplan?: boolean
}
```

#### Plan File Management

**`eD(agentId)` — plan file path:**
```javascript
jO() + "/" + hF(sessionId) + ".md"                        // main session
jO() + "/" + hF(sessionId) + "-agent-" + agentId + ".md" // sub-agent
```
`jO()` = plan files base directory; `hF()` = generates/caches a random name per session.

**`sM(agentId)`** — reads plan file content. Returns `null` if not found (`ENOENT` → `null`).

**Plan file recovery (`FW1()`):** If plan file is missing during resume:
1. Tries to recover from file snapshot in messages (`x8Y()`).
2. Falls back to scanning message history (`b8Y()`).
3. Writes recovered content back to disk.

#### Implementation Logic

1. **Teammate mode (`Oz() && Dp6()`):** Sends plan approval request to team lead via `z9("team-lead", ...)`, sets `awaitingPlanApproval` flag, returns `{awaitingLeaderApproval: true}`. Throws if no plan file exists yet.

2. **Normal mode:**
   - Reads plan content from `sM(agentId)`.
   - Transitions permission mode back to pre-plan mode via `setAppState`: `prePlanMode → mode`.
   - If `prePlanMode === "ultraplan"` → `mode = "default"`.
   - If `prePlanMode === "auto"` and `isAutoModeGateEnabled()` → restores auto mode.
   - Calls `yk(!0)` (sets plan file written flag), `eh(!0)` (sets exited flag).
   - Returns `{plan, isAgent, filePath, hasTaskTool, isUltraplan}`.

#### Tool Result Messages (to model)

| Condition | Message |
|-----------|----------|
| Sub-agent / teammate | `'User has approved the plan. There is nothing else needed from you now. Please respond with "ok"'` |
| Awaiting team lead | `"Your plan has been submitted to the team lead for approval... Do NOT proceed until you receive approval."` |
| No plan content | `"User has approved exiting plan mode. You can now proceed."` |
| Ultraplan session | `"User has reviewed the ultraplan. There is nothing else to do. Respond with a brief summary."` |
| Normal + Agent tool available | Appends suggestion to use Agent tool for parallelization |
| Normal with plan | `"User has approved your plan... Your plan has been saved to: {path}... ## Approved Plan:\n{plan}"` |

#### Permission Model

- **Teammates (`Oz()`):** Always `{behavior: "allow"}`.
- **Interactive users:** `{behavior: "ask", message: "Exit plan mode?"}`.
- `requiresUserInteraction()`: `!Oz()` — only interactive for non-teammates.

#### Plan Mode Reminder System

`a4Y(messages, context)` — generates plan_mode attachment every turn:
- Skips if fewer than `TURNS_BETWEEN_ATTACHMENTS` (5) turns since last attachment.
- Alternates between `"full"` and `"sparse"` reminder types (full every 5th attachment).
- `"ultraplan-complete"` type for ultraplan sessions.
- Includes `planFilePath` and `planExists` in attachment.
- Re-entry reminder (`"plan_mode_reentry"`) if coming back after disconnect.

---

## 8. Extensibility Tools

Five tools that extend Claude's capabilities with skills, tool discovery, task tracking, language server protocol, and worktrees. All declare `shouldDefer: true` and `maxResultSizeChars: 100,000`.

---

### Skill

**Tool name:** `"Skill"`  
**Constant:** `var nj = "Skill"` (line 250370)  
**shouldDefer:** `true`  
**maxResultSizeChars:** `1e5`

#### Input Schema

```typescript
{
  skill: string
    // Skill name — e.g., "commit", "review-pr", "pdf", "ms-office-suite:pdf"
    // Leading "/" is stripped if present

  args?: string
    // Optional arguments for the skill
}
```

#### Output Schema (discriminated union)

```typescript
// Inline execution:
{
  success: boolean
  commandName: string
  allowedTools?: string[]
  model?: string
  status?: "inline"
}

// Forked execution:
{
  success: boolean
  commandName: string
  status: "forked"
  agentId: string
  result: string
}
```

#### Validation Error Codes

| Code | Meaning |
|------|---------|
| 1 | Empty skill name |
| 2 | Unknown skill |
| 3 | Cannot load skill |
| 4 | `disableModelInvocation` is set |
| 5 | Not a prompt-based skill |

#### Implementation Logic

1. Trims skill name, strips leading `/` if present.
2. Checks `sQ(name, commands)` for existence; checks `Tu(name, commands)` can load it.
3. **Fork vs inline:** If `skill.context === "fork"`, calls `b8z()` (spawns sub-agent via `xC()`); otherwise runs inline.
4. **Inline path:** Calls `processPromptSlashCommand`, gets `messages`, `allowedTools`, `model`; injects messages as `newMessages` into current context; hooks registered via `fL1()`.
5. **Tool result:** Returns `"Launching skill: <name>"` for inline; `"Skill \"<name>\" completed (forked execution).\nResult:\n<result>"` for forked.
6. **Analytics:** Emits `tengu_skill_tool_invocation` event with `command_name` and plugin info.

#### Permission Model

- Checks deny rules first (`Cu(context, tool, "deny")`), then allow rules.
- Rule matching: exact name OR prefix wildcard (`skill:*`).
- If skill has no non-standard frontmatter (`m8z(skill)`), auto-allows.
- Otherwise: `{behavior: "ask", message: "Execute skill: <name>", suggestions: [...addRules...]}`.
- Suggestions offer two options: allow exact name, or allow `name:*` wildcard.

#### Budget Constants (lines 250490–250494)

```javascript
SKILL_BUDGET_CONTEXT_PERCENT = b24 = 0.02  // 2% of context window
CHARS_PER_TOKEN              = x24 = 4
DEFAULT_CHAR_BUDGET          = u24 = 16000 // chars
MIN_DESC_CHARS               = kAY = 20
```

**`qZ1(contextTokens)`** — character budget for skill listing:
- If `SLASH_COMMAND_TOOL_CHAR_BUDGET` env var is set: use that value.
- If `contextTokens` provided: `floor(contextTokens * 4 * 0.02)`.
- Default: 16,000 chars.

**`FE8(commands, contextTokens)`** — formats skill list within budget:
- Bundled skills get full description; custom skills get truncated descriptions.
- If per-skill budget < 20 chars: show only names for custom skills.

---

### ToolSearch

**Tool name:** `"ToolSearch"`  
**Constant:** `var OW = "ToolSearch"` (line 249979)  
ToolSearch itself **never defers** — it is always loaded and available to discover other deferred tools.

#### Input Schema

```typescript
{
  query: string
    // "select:<tool_name>" for direct selection, or keywords for search
    // Comma-separated: "select:A,B,C" loads multiple tools at once

  max_results?: number
    // default: 5
}
```

#### Output Schema

```typescript
{
  matches: string[]              // Tool names of matched tools
  query: string
  total_deferred_tools: number
  pending_mcp_servers?: string[] // Only present if > 0
}
```

#### Ranking Algorithm (`TAY(query, deferredTools, allTools, maxResults)`, lines ~250127–250188)

1. **Exact name match** — returns `[toolName]` immediately.
2. **`mcp__` prefix match** — filters all tools starting with that prefix, returns up to `maxResults`.
3. **`+keyword` required filter** — restricts candidates to tools matching the required keyword(s).
4. **Scoring per tool** (each query term scored against each tool):
   - `parts.includes(term)` exact part match: **+10** (non-MCP) / **+12** (MCP)
   - `parts.some(p => p.includes(term))` substring in part: **+5** (non-MCP) / **+6** (MCP)
   - `full.includes(term)` and score === 0: **+3**
   - `searchHint && wordBoundary(searchHint, term)`: **+4**
   - `description && wordBoundary(description, term)`: **+2**
5. Filters `score > 0`, sorts descending, slices to `maxResults`.

**Tool name parsing (`L24`):**
- MCP tools (`mcp__X__Y`): strips prefix, splits on `__` then `_` → parts array.
- Non-MCP tools: camelCase split + `_` split.

#### Select Mode

Direct `select:<tool_name>` or comma-separated `select:A,B,C` — calls `TAY` with exact name matching first.

#### Deferred Tool Detection (`GG`)

```javascript
// Tool is deferred if ANY of these are true:
isMcp === true                    // MCP tools always defer
name === OW (ToolSearch)          // ToolSearch itself: NEVER deferred
featureFlag("tengu_defer_all_bn4") // Defers all tools when enabled
shouldDefer === true              // Explicit opt-in to deferral
```

#### Description Caching

`sW1` is a memoized async function keyed by `(toolName, allToolsKey)`. Cache key `ZAY()` is a sorted comma-joined string of all tool names. Public cache clear function: `fAY()`. Cache invalidated when deferred tool list changes.

---

### TodoWrite

**Tool name:** `"TodoWrite"`  
**Constant:** `var HF = "TodoWrite"` (line 231490)  
**strict:** `true`  
**shouldDefer:** `true`  
**maxResultSizeChars:** `1e5`  
**isEnabled:** `() => !iH()` — note: disabled when tasks feature is active (the two systems are mutually exclusive)  
**isConcurrencySafe:** `() => false`  
**isReadOnly:** `() => false`

#### Input Schema

```typescript
I.strictObject({
  todos: Array<{
    content: string    // min length 1 — noun form, e.g. "Fix the login bug"
    status: "pending" | "in_progress" | "completed"
    activeForm: string // min length 1 — gerund form, e.g. "Fixing the login bug"
  }>
  // description: "The updated todo list"
})
```

#### Output Schema

```typescript
{
  oldTodos: Todo[]      // List before update
  newTodos: Todo[]      // List after update
  verificationNudgeNeeded?: boolean
}
```

#### Task State Rules (from prompt, lines ~231435–231490)

- Exactly ONE task must be `in_progress` at any time.
- ONLY mark `completed` when FULLY accomplished.
- Complete current tasks before starting new ones.
- If blocked, create a new task describing what needs to be resolved.
- Mark tasks complete IMMEDIATELY after finishing (no batch completions).
- Remove no-longer-relevant tasks entirely.
- Always provide both `content` (noun form) and `activeForm` (gerund form).

#### Implementation Logic

1. Reads current todos from app state keyed by `agentId ?? sessionId`.
2. If every todo has `status === "completed"`: saves empty array (auto-clears completed list).
3. Updates app state with new todos.
4. Returns `{oldTodos, newTodos, verificationNudgeNeeded: false}`.
5. **Tool result message:** `"Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable"`.
6. If `verificationNudgeNeeded` (3+ tasks closed without a verification step): appends a note prompting to spawn a verification agent with `subagent_type` = `rUA`.

#### Permission Model

`checkPermissions`: always `{behavior: "allow"}` — no user confirmation needed.

---

### LSP

**Tool name:** `"LSP"`  
**Constant:** `var Aa6 = "LSP"` (line 446706)  
**shouldDefer:** `true`  
**maxResultSizeChars:** `1e5`  
**isConcurrencySafe:** `() => true`  
**isReadOnly:** `() => true`

#### Input Schema — Discriminated Union on `operation`

All 9 variants share `filePath`, `line`, and `character` (all 1-based positive integers). Discriminated by the `operation` literal field. All use `I.strictObject`.

```typescript
type LSPInput =
  | { operation: "goToDefinition";       filePath: string; line: int; character: int }
  | { operation: "findReferences";       filePath: string; line: int; character: int }
  | { operation: "hover";                filePath: string; line: int; character: int }
  | { operation: "documentSymbol";       filePath: string; line: int; character: int }
  | { operation: "workspaceSymbol";      filePath: string; line: int; character: int }
  | { operation: "goToImplementation";   filePath: string; line: int; character: int }
  | { operation: "prepareCallHierarchy"; filePath: string; line: int; character: int }
  | { operation: "incomingCalls";        filePath: string; line: int; character: int }
  | { operation: "outgoingCalls";        filePath: string; line: int; character: int }
```

`line` and `character` use `number().int().positive()` validation.

#### Output Schema

```typescript
{
  operation: "goToDefinition" | "findReferences" | "hover" | "documentSymbol" |
             "workspaceSymbol" | "goToImplementation" | "prepareCallHierarchy" |
             "incomingCalls" | "outgoingCalls"
  result: string       // Formatted result text
  filePath: string
  resultCount?: number // int, non-negative
  fileCount?: number   // int, non-negative
}
```

#### LSP Method Mappings (`c9z` function)

| `operation` | LSP Protocol Method | Notes |
|-------------|---------------------|-------|
| `goToDefinition` | `textDocument/definition` | |
| `findReferences` | `textDocument/references` | `includeDeclaration: true` |
| `hover` | `textDocument/hover` | |
| `documentSymbol` | `textDocument/documentSymbol` | |
| `workspaceSymbol` | `workspace/symbol` | `query: ""` |
| `goToImplementation` | `textDocument/implementation` | |
| `prepareCallHierarchy` | `textDocument/prepareCallHierarchy` | |
| `incomingCalls` | `textDocument/prepareCallHierarchy` → `callHierarchy/incomingCalls` | Two-step |
| `outgoingCalls` | `textDocument/prepareCallHierarchy` → `callHierarchy/outgoingCalls` | Two-step |

`incomingCalls` and `outgoingCalls` are two-step operations: first `prepareCallHierarchy` is called, then the actual calls operation is performed on the first result item.

#### Enabled Condition

```javascript
isEnabled() {
  if (n26().status === "failed") return false;
  const manager = dn();
  if (!manager) return false;
  const servers = manager.getAllServers();
  if (servers.size === 0) return false;
  return Array.from(servers.values()).some(s => s.state !== "error");
}
```

The tool is disabled if the LSP subsystem failed to initialize, no LSP manager exists, no servers are registered, or all servers are in error state.

#### Implementation Logic

1. **Validation:** Resolves absolute path, checks file exists via `stat()`. Error codes: `1` = ENOENT, `2` = not a file, `3` = schema invalid, `4` = stat failed.
2. Waits for LSP init if `n26().status === "pending"` via `await z9q()`.
3. Opens file if not already open: reads content and calls `w.openFile(K, D)`.
4. Sends LSP request: `w.sendRequest(file, method, params)`.
5. If `undefined` returned: no LSP server for that file type → returns descriptive error.
6. **Workspace filtering** (for `findReferences`, `goToDefinition`, `goToImplementation`, `workspaceSymbol`): filters results to only include files within workspace via `q$q(locations, workspaceRoot)`.
7. **workspaceSymbol:** Filters `SymbolInformation` by location URI against workspace root.
8. Formats results via `n9z(operation, rawResult, workspaceRoot)` → `{formatted, resultCount, fileCount}`.

#### Permission Model

Uses `U66(tc8, input, context)` — standard read-only permission check.

#### Symbol Kind Mapping (`Ak6`)

Full LSP SymbolKind numeric mapping: 1=File, 2=Module, 3=Namespace, 4=Package, 5=Class, 6=Method, 7=Property, 8=Field, 9=Constructor, 10=Enum, 11=Interface, 12=Function, 13=Variable, 14=Constant, 15=String, 16=Number, 17=Boolean, 18=Array, 19=Object, 20=Key, 21=Null, 22=EnumMember, 23=Struct, 24=Event, 25=Operator, 26=TypeParameter.

#### Symbol Extraction for Display (`i_q`)

Buffer limit `l_q = 65,536` bytes. Reads file synchronously up to 64 KB, finds word/operator token at `(line-1, char-1)`. Used in `renderToolUseMessage` to produce display like: `"operation: \"goToDefinition\", symbol: \"myFunc\", in: \"path/to/file\""`.

---

### EnterWorktree

**Tool name:** `"EnterWorktree"`  
**Constant:** `var FT1 = "EnterWorktree"` (line 290075)  
**shouldDefer:** `true`  
**isConcurrencySafe:** `() => false`  
**isReadOnly:** `() => false`  
Member of `QT1` (allowed in async mode).

#### Input Schema

```typescript
I.strictObject({
  name?: string
    // Optional worktree name; randomly generated if omitted
})
```

#### Output Schema

```typescript
{
  worktreePath: string
  worktreeBranch?: string
  message: string
}
```

#### Implementation Logic

1. Throws `"Already in a worktree session"` if `hL()` returns true (already in worktree).
2. Resolves project root: `y0(I1())` — if different from current `I1()`, calls `chdir` and `rH()` to normalize.
3. Generates name: `A.name ?? hF()` (random if not provided).
4. Calls `$o6(sessionId, name)` — creates the worktree:
   - **In git repo:** Creates `git worktree add .claude/worktrees/<name> -b <branch>` based on HEAD.
   - **Outside git repo:** Delegates to `WorktreeCreate`/`WorktreeRemove` hooks from `settings.json`.
5. `chdir` to new worktree path, updates `rH()`, `v46()`, `fR6(!0)`, `GT1()`, clears `lH.cache` and `jO.cache`.
6. Fires `tengu_worktree_created` analytics event with `mid_session: true`.
7. Returns `{worktreePath, worktreeBranch, message}`.
8. **Tool result to model:** The message string directly.

#### Permission Model

`checkPermissions`: always `{behavior: "allow"}` — no user confirmation.

#### When to Use

Only when user **explicitly** says "worktree". NOT for branch creation, feature work, or bug fixes unless the user specifically mentions "worktree".

#### Display

`"Switched to worktree on branch <bold>branchName</bold>"` + `"<path>"` in dimmed text.

---

## Cross-Cutting Reference

### Tool Visibility Sets

| Set | Members | Effect |
|-----|---------|--------|
| `cT6` | `TaskOutput (SI)`, `ExitPlanMode (aM)`, `EnterPlanMode (p16)`, `Agent (Tq)`, `AskUserQuestion (b_)`, `TaskStop (RI)` | Hidden from tool list by default |
| `eC8` | Same as `cT6` | Hidden from non-built-in tool sources |
| `QT1` | `TodoWrite (HF)`, `WebSearch (tV)`, `WebFetch (VM)`, `Skill (nj)`, `ToolSearch (OW)`, `EnterWorktree (FT1)`, plus others | Allowed in async/background mode |
| `Gf4` | `TaskCreate (US)`, `TaskGet (c16)`, `TaskList (l16)`, `TaskUpdate (KL)`, plus others | Async set for in-process teammates (requires `Z7() && AW()`) |

### Feature Gate Summary

| Tool | `isEnabled()` Gate |
|------|--------------------|
| Agent | Always available |
| TaskCreate | `iH()` — interactive OR `CLAUDE_CODE_ENABLE_TASKS=1` |
| TaskGet | `iH()` |
| TaskUpdate | `iH()` |
| TaskList | `iH()` |
| TaskOutput | Always enabled |
| TaskStop | Always enabled |
| WebFetch | Always enabled |
| WebSearch | firstParty/foundry always; Vertex only on claude-4 family; otherwise disabled |
| TodoWrite | `!iH()` — disabled when tasks feature is active |
| LSP | At least one LSP server registered and not in error state |
| Skill | Always (when skills are configured) |
| ToolSearch | Always |
| AskUserQuestion | Always (but denied in auto mode at permission layer) |
| EnterPlanMode | Always (but throws in agent contexts at runtime) |
| ExitPlanMode | Always |
| EnterWorktree | Always |

### `shouldDefer` Tools

Tools with `shouldDefer: true` are not loaded into the model's tool list by default. They must be discovered via ToolSearch and loaded on demand:
`Skill`, `AskUserQuestion`, `EnterPlanMode`, `ExitPlanMode`, `EnterWorktree`, `LSP`, `TodoWrite`, `WebFetch`, `WebSearch`, `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList`, `TaskOutput`, `TaskStop`.

ToolSearch itself **never defers** — it is always available to discover other tools.
## 9. Scheduling Tools

The three scheduling tools (`CronCreate`, `CronDelete`, `CronList`) implement a session-scoped cron scheduler that fires prompts into the REPL when it is idle. They are registered unconditionally in the `Nzz` array and included in `YU()` via `...Nzz`, but each tool gates on the `lC()` feature flag at `isEnabled()` time.

**Feature flag:** `lC()` calls `jU("tengu_kairos_cron", false, 300000)` — Statsig/GrowthBook gate `tengu_kairos_cron`, 5-minute (300,000 ms) TTL cache. **Disabled by default.**

**Registration (lines in `mP` module initializer):**
```javascript
Nzz = [
  (AHq(), W3(eOq)).CronCreateTool,
  (KHq(), W3(qHq)).CronDeleteTool,
  (zHq(), W3(YHq)).CronListTool,
]
```

---

### CronCreate

**Variable name constant:** `HU = "CronCreate"`

**Description:** Schedule a prompt to be enqueued into the REPL at a cron-defined time. Supports both recurring (default) and one-shot execution.

**Input Schema** (Zod `I.strictObject`; the model-facing `inputSchema` is `dYz()` which omits `durable`):

```typescript
{
  cron: string;
    // Standard 5-field cron expression in local time: "M H DoM Mon DoW"
    // Examples: "*/5 * * * *" = every 5 minutes
    //           "30 14 28 2 *" = Feb 28 at 2:30pm local once

  prompt: string;
    // The prompt to enqueue at each fire time.

  recurring?: boolean;
    // true (default) = fire on every cron match until deleted or auto-expired
    //   after 3 days.
    // false = fire once at next match, then auto-delete.

  // NOTE: 'durable' exists in internal schema (pYz) but is NOT exposed
  // in the model-facing inputSchema (dYz):
  // durable?: boolean;
  //   true  = persist to .claude/scheduled_tasks.json, survives restarts.
  //   false = in-memory only (default), dies when Claude session ends.
}
```

**Output Schema:**
```typescript
{
  id: string;              // 8-character UUID prefix
  humanSchedule: string;  // Human-readable description (e.g., "Every 5 minutes")
  recurring: boolean;
  durable?: boolean;
}
```

**Tool Properties:**

| Property | Value |
|---|---|
| `name` | `"CronCreate"` |
| `searchHint` | `"schedule a recurring prompt for this session"` |
| `maxResultSizeChars` | `100000` |
| `shouldDefer` | `true` |
| `isConcurrencySafe()` | `false` |
| `isReadOnly()` | `false` |
| `checkPermissions()` | Always `{ behavior: "allow", updatedInput: A }` |

**validateInput — three checks (in order):**
1. `_a6(A.cron)` — validates 5-field cron syntax. Fields: minute (0–59), hour (0–23), day-of-month (1–31), month (1–12), day-of-week (0–6). Error code 1.
2. `wk6(A.cron, Date.now())` — confirms a next-fire time exists within 1 year (searches up to 527,040 minutes = 366 × 24 × 60). If null → error `"does not match any calendar date in the next year"`, error code 2.
3. `(await _k6()).length >= 50` → error `"Too many scheduled jobs (max 50). Cancel one first."` Limit constant: `tOq = 50`. Error code 3.

**call() implementation:**
```javascript
async call({ cron, prompt, recurring = true, durable = false }) {
  let id = await UOq(cron, prompt, recurring, durable);
  XR6(true); // sets u1.scheduledTasksEnabled = true
  return { data: { id, humanSchedule: zk6(cron), recurring, durable } };
}
```

**UOq internals:** Generates 8-char UUID prefix as job ID. For `durable: false` (default): calls `_F1(job)` → pushes to `u1.sessionCronTasks` (in-memory). For `durable: true`: reads `.claude/scheduled_tasks.json` via `Oa6()`, appends, writes back via `QOq()`.

**Scheduler mechanics:**
- `zk6(cron)` — produces human-readable schedule string (e.g., `"Every 5 minutes"`, `"Every day at 9:03 AM"`)
- `wk6(cron, now)` = `FOq(parsedCron, new Date(now))` — iterates up to 527,040 minutes forward
- **Jitter system:**
  - Recurring tasks fire up to 10% of their period late, capped at 15 minutes (`recurringCapMs = 900000`)
  - One-shot tasks scheduled on `:00` or `:30` fire 0–90 seconds early (`oneShotMaxMs = 90000`, `oneShotMinuteMod = 30`)
- **Auto-expiry:** Recurring tasks expire 3 days after creation (checked by `XQq(F, R)`)
- **Idle guard:** Scheduler tick `v()` only fires when REPL is not mid-query: checks `H?.()` (isRunning guard) and `K() && !Y` (idle check)

**mapToolResultToToolResultBlockParam result text:**
```
Recurring: "Scheduled recurring job {id} ({humanSchedule}). {persistence}.
           Auto-expires after 3 days. Use CronDelete to cancel sooner."
One-shot:  "Scheduled one-shot task {id} ({humanSchedule}). {persistence}.
           It will fire once then auto-delete."

Persistence (durable=true):  "Persisted to .claude/scheduled_tasks.json"
Persistence (durable=false): "Session-only (not written to disk, dies when Claude exits)"
```

---

### CronDelete

**Variable name constant:** `H_6 = "CronDelete"`

**Description:** Cancel a scheduled job by its ID. Removes both in-memory and durable jobs.

**Input Schema** (Zod `I.strictObject`):
```typescript
{
  id: string;  // Job ID returned by CronCreate.
}
```

**Output Schema:**
```typescript
{
  id: string;  // The deleted job ID.
}
```

**Tool Properties:**

| Property | Value |
|---|---|
| `isConcurrencySafe()` | `false` |
| `isReadOnly()` | `false` |
| `shouldDefer` | `true` |
| `checkPermissions()` | Always `{ behavior: "allow" }` |

**validateInput:** Calls `_k6()` to get all jobs (merges durable + session-only). If no job matches `A.id`: error `"No scheduled job with id '{id}'"`, error code 1.

**`_k6()` logic:** Merges durable jobs from `.claude/scheduled_tasks.json` (via `Oa6()`) with session-only jobs from `PR6()` (= `u1.sessionCronTasks`), tagging session ones as `{ durable: false }`.

**call() implementation:**
```javascript
async call({ id }) {
  await Ha6([id]); // removes from disk and/or in-memory store
  return { data: { id } };
}
```

**Ha6 internals:** If all removed IDs are in-memory only (`WR6(A) === A.length`), skips disk write. Otherwise reads durable tasks, filters out matching IDs, writes back. `WR6()` handles in-memory removal by mutating `u1.sessionCronTasks`.

**mapToolResultToToolResultBlockParam result text:**
```
"Cancelled job {id}."
```

---

### CronList

**Variable name constant:** `RS1 = "CronList"`

**Description:** List all scheduled jobs for the current session, including both in-memory and durable jobs.

**Input Schema** (Zod `I.strictObject`):
```typescript
{}  // No parameters.
```

**Output Schema:**
```typescript
{
  jobs: Array<{
    id: string;
    cron: string;
    humanSchedule: string;
    prompt: string;
    recurring?: boolean;
    durable?: boolean;
  }>;
}
```

**Tool Properties:**

| Property | Value |
|---|---|
| `isConcurrencySafe()` | `true` (only cron tool that is concurrency-safe) |
| `isReadOnly()` | `true` |
| `shouldDefer` | `true` |
| `checkPermissions()` | Always `{ behavior: "allow" }` |

**call() implementation:**
```javascript
async call() {
  return {
    data: {
      jobs: (await _k6()).map((K) => ({
        id: K.id,
        cron: K.cron,
        humanSchedule: zk6(K.cron),
        prompt: K.prompt,
        ...(K.recurring ? { recurring: true } : {}),
        ...(K.durable === false ? { durable: false } : {}),
      })),
    },
  };
}
```

**mapToolResultToToolResultBlockParam result text:**
- If no jobs: `"No scheduled jobs."`
- Otherwise: one line per job:
  ```
  "{id} — {humanSchedule} (recurring|one-shot)[session-only]: {prompt truncated to 80 chars}"
  ```

---

## 10. Team Communication

### SendMessage

**Variable name constant:** `Gx = "SendMessage"`

**Description:** Send messages between agent teammates in a swarm (multi-agent team). Supports direct messages, broadcasts, shutdown negotiation, and plan approval workflows.

**Feature flag:** `Z7()` — requires EITHER env var `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` OR CLI flag `--agent-teams`, AND Statsig gate `tengu_amber_flint` must be `true`. **Disabled by default.**

**Registration in `YU()`:**
```javascript
...(Z7() ? [vzz(), kzz(), Ezz()] : [])  // TeamCreate, TeamDelete, SendMessage
```
`Ezz = () => (xHq(), W3(bHq)).SendMessageTool`

**Input Schema — Discriminated Union** (Zod `I.discriminatedUnion("type", [...])`, exposed as `Jzz()`):

```typescript
// Variant 1: Direct message
{
  type: "message";
  recipient: string;    // Agent name of the target
  content: string;      // Message body
  summary: string;      // 5–10 word preview shown in UI
}

// Variant 2: Broadcast to all teammates
{
  type: "broadcast";
  content: string;
  summary: string;      // 5–10 word preview shown in UI
  // No recipient — sent to all team members except self
}

// Variant 3: Request a teammate to shut down
{
  type: "shutdown_request";
  recipient: string;
  content?: string;     // Optional reason
}

// Variant 4: Respond to a shutdown request
{
  type: "shutdown_response";
  request_id: string;   // From the original shutdown_request result
  approve: boolean;
  content?: string;     // Required when approve=false (enforced in validateInput)
}

// Variant 5: Respond to a plan submitted by a teammate
{
  type: "plan_approval_response";
  request_id: string;
  approve: boolean;
  recipient: string;    // The teammate who submitted the plan
  content?: string;     // Feedback when approve=false
}
```

**Also has `inputJSONSchema`** (raw JSON Schema, bypasses Zod — used by MCP serialization path):
```json
{
  "type": "object",
  "properties": {
    "type": {
      "type": "string",
      "enum": ["message", "broadcast", "shutdown_request", "shutdown_response", "plan_approval_response"]
    },
    "recipient": {
      "type": "string",
      "description": "Agent name of recipient (required for message, shutdown_request, plan_approval_response)"
    },
    "content": { "type": "string", "description": "Message text, reason, or feedback" },
    "summary": {
      "type": "string",
      "description": "5-10 word summary shown as preview in UI (required for message, broadcast)"
    },
    "request_id": {
      "type": "string",
      "description": "Request ID to respond to (required for shutdown_response, plan_approval_response)"
    },
    "approve": {
      "type": "boolean",
      "description": "Whether to approve the request (required for shutdown_response, plan_approval_response)"
    }
  },
  "required": ["type"],
  "additionalProperties": false
}
```

**Tool Properties:**

| Property | Value |
|---|---|
| `name` | `"SendMessage"` |
| `searchHint` | `"send messages to agent teammates (swarm protocol)"` |
| `maxResultSizeChars` | `100000` |
| `shouldDefer` | `true` |
| `isConcurrencySafe()` | `false` |
| `isReadOnly()` | `true` only for `type === "message"` or `type === "broadcast"` |
| `checkPermissions()` | Always `{ behavior: "allow" }` |

**validateInput:**
1. If `recipient` is present but empty string → error `"recipient must not be empty"`, error code 9
2. If `type === "shutdown_response"` AND `approve === false` AND no/empty `content` → error `"content (reason) is required when rejecting a shutdown request"`, error code 9

**call() dispatch table:**
```javascript
switch (A.type) {
  case "message":           return Dzz(A, q);
  case "broadcast":         return Xzz(A, q);
  case "shutdown_request":  return Pzz(A, q);
  case "shutdown_response":
    if (A.approve) return Wzz(A, q);  // approve
    return Zzz(A);                     // reject
  case "plan_approval_response":
    if (A.approve) return Gzz(A, q);  // approve
    return fzz(A, q);                  // reject
}
```

**Per-variant implementation details:**

**`Dzz` — `type: "message"`:**
- Resolves sender: `V9() || (Oz() ? "teammate" : "team-lead")` (checks `CLAUDE_CODE_AGENT_NAME` env)
- Strips `@` prefix from recipient if present
- Calls `z9(recipient, { from, text, summary, timestamp, color }, teamName)` to deliver to agent inbox
- Returns: `{ data: { success: true, message: "Message sent to {name}'s inbox", routing: { sender, senderColor, target, targetColor, summary, content } } }`

**`Xzz` — `type: "broadcast"`:**
- Requires team context; loads team from `Q26(teamName)` (reads `~/.claude/teams/{team-name}.json`)
- Skips self (case-insensitive name match)
- Calls `z9()` for each other `team.members` entry
- Returns: `{ data: { success: true, message: "Message broadcast to N teammate(s): name1, name2", recipients: [...] } }`
- Edge case (only member): `{ data: { success: true, message: "No teammates to broadcast to...", recipients: [] } }`

**`Pzz` — `type: "shutdown_request"`:**
- Generates request ID: `Yf6("shutdown", recipient)` → `"shutdown-{Date.now()}@{recipient}"`
- Sends structured JSON via `z9()`: `Hf6({ requestId, from, reason })`
- Returns: `{ data: { success: true, message: "Shutdown request sent to {name}. Request ID: {id}", request_id, target } }`

**`Wzz` — `type: "shutdown_response"`, `approve: true`:**
- Finds own entry in team members to get `tmuxPaneId` and `backendType`
- Sends approval `JL8({ requestId, from, paneId, backendType })` to `"team-lead"` via `z9()`
- If `backendType === "in-process"`: aborts own `abortController` found in AppState tasks
- Otherwise: calls `setImmediate(() => $K(0, "other"))` to exit the process
- Returns: `{ data: { success: true, message: "Shutdown approved. Sent confirmation to team-lead. Agent {name} is now exiting.", request_id } }`

**`Zzz` — `type: "shutdown_response"`, `approve: false`:**
- Sends rejection `ML8({ requestId, from, reason })` to `"team-lead"` via `z9()`
- Returns: `{ data: { success: true, message: "Shutdown rejected. Reason: '...'. Continuing to work.", request_id } }`

**`Gzz` — `type: "plan_approval_response"`, `approve: true`:**
- Guard: `JG(K.teamContext)` must be true; throws `"Only the team lead can approve plans."`
- Gets current permission mode, maps `"plan"` → `"default"`
- Sends: `{ type: "plan_approval_response", requestId, approved: true, timestamp, permissionMode }` to recipient
- Returns: `{ data: { success: true, message: "Plan approved for {recipient}.", request_id } }`

**`fzz` — `type: "plan_approval_response"`, `approve: false`:**
- Same team-lead guard as `Gzz`
- `feedback = A.content || "Plan needs revision"`
- Sends: `{ type: "plan_approval_response", requestId, approved: false, feedback, timestamp }` to recipient
- Returns: `{ data: { success: true, message: "Plan rejected for {recipient} with feedback: '...'", request_id } }`

**mapToolResultToToolResultBlockParam:** Returns full JSON-stringified output (no abbreviation):
```javascript
{
  tool_use_id: q,
  type: "tool_result",
  content: [{ type: "text", text: JSON.stringify(A, null, 2) }],
}
```

**renderToolResultMessage (`CHq`):**
- Parses result (JSON if string)
- If `routing` property present: returns `null` (routing messages rendered separately in UI)
- If `request_id` and `target` present: returns `null`
- Otherwise: renders `A.message` in dimColor text

---

## 11. Browser & Computer Use Tools

The 18 browser/computer use tools enable Claude to control a Chrome browser through a native socket or WebSocket bridge connection to the Claude browser extension. They are defined in a single lazy-initialized module (`Al1` function, lines 18651–19191) and stored in module-level variable `ip`.

**Export at line 30870:** `BROWSER_TOOLS: () => ip`

**MCP server:** Created by `$41()` (`createClaudeForChromeMcpServer`) at line 30822.

> **Note:** These tools are served by the "Claude in Chrome" MCP server and are distinct from `mcp__chrome-devtools__*` tools, which connect to Chrome via the DevTools Protocol (CDP). The two systems use different transports and have independent scopes.

---

### 1. `javascript_tool`

**Description:** Execute JavaScript code in the context of the current page.

**Input Schema:**
```typescript
{
  action: string;   // Must be set to 'javascript_exec'
  text: string;     // The JavaScript code to execute. No 'return' statements;
                    // write the expression directly (e.g., 'window.myData.value')
  tabId: number;    // Tab ID to execute in. Must be in current group.
}
// required: ["action", "text", "tabId"]
```

**Output:** Returns the result of the last expression or any thrown errors.

**Key behaviors:**
- Code runs in page context with access to DOM, `window` object, and all page variables

---

### 2. `read_page`

**Description:** Get an accessibility tree of elements on the page.

**Input Schema:**
```typescript
{
  tabId: number;            // Tab ID to read from. (required)
  filter?: "interactive" | "all";
                            // "interactive" returns buttons/links/inputs only.
                            // "all" (default) includes non-visible elements.
  depth?: number;           // Maximum tree depth to traverse. Default: 15.
  ref_id?: string;          // Reference ID of a parent element;
                            // returns that element and its children only.
  max_chars?: number;       // Maximum characters for output. Default: 50000.
}
// required: ["tabId"]
```

**Output:** Accessibility tree text representation.

**Key behaviors:**
- Default (`filter: "all"`) includes non-visible elements
- If output exceeds `max_chars` (default 50,000), returns error asking caller to use smaller `depth` or a specific `ref_id`

---

### 3. `find`

**Description:** Find elements on the page using natural language.

**Input Schema:**
```typescript
{
  query: string;   // Natural language description.
                   // Examples: "search bar", "add to cart button",
                   //           "product title containing organic"
  tabId: number;   // Tab ID to search in.
}
// required: ["query", "tabId"]
```

**Output:** Up to 20 matching elements with reference IDs usable by other tools.

**Key behaviors:**
- Returns a maximum of 20 matches; if more exist, notifies caller to use a more specific query

---

### 4. `form_input`

**Description:** Set values in form elements using a reference ID from `read_page`.

**Input Schema:**
```typescript
{
  ref: string;                       // Element reference ID from read_page
                                     // (e.g., "ref_1", "ref_2")
  value: string | boolean | number;  // For checkboxes: boolean.
                                     // For selects: option value or text.
                                     // For other inputs: string or number.
  tabId: number;                     // Tab ID to set form value in.
}
// required: ["ref", "value", "tabId"]
```

**Output:** Text confirmation that the form value was set.

---

### 5. `computer`

**Description:** Mouse and keyboard interaction with a web browser, plus screenshots.

**Input Schema:**
```typescript
{
  action: "left_click" | "right_click" | "type" | "screenshot" | "wait"
        | "scroll" | "key" | "left_click_drag" | "double_click" | "triple_click"
        | "zoom" | "scroll_to" | "hover";
  tabId: number;                          // Tab ID for the action. (required)

  coordinate?: [number, number];          // [x, y] pixels from left/top edge.
                                          // Required for: left_click, right_click,
                                          // double_click, triple_click, scroll.
                                          // For left_click_drag: the end position.
  text?: string;                          // Text to type ("type" action) or
                                          // space-separated keys ("key" action).
                                          // Supports "cmd+a", "ctrl+a" combos.
  duration?: number;                      // Seconds to wait. Required for "wait".
                                          // Range: 0–30.
  scroll_direction?: "up" | "down" | "left" | "right";
                                          // Required for "scroll".
  scroll_amount?: number;                 // Scroll wheel ticks. Range: 1–10.
                                          // Default: 3.
  start_coordinate?: [number, number];    // Starting coordinates for
                                          // "left_click_drag".
  region?: [number, number, number, number];
                                          // [x0, y0, x1, y1] rectangular region.
                                          // Required for "zoom".
  repeat?: number;                        // Times to repeat key sequence.
                                          // Only for "key". Range: 1–100. Default: 1.
  ref?: string;                           // Element reference ID from read_page/find.
                                          // Required for "scroll_to";
                                          // alternative to coordinate for clicks.
  modifiers?: string;                     // Modifier keys: "ctrl", "shift", "alt",
                                          // "cmd"/"meta", "win"/"windows".
                                          // Combinable with "+" (e.g., "ctrl+shift").
}
// required: ["action", "tabId"]
```

**Output:**
- For `screenshot` and `zoom`: Image content block (`type: "image"`, base64 data, `mimeType: "image/png"`). Screenshots generate an `imageId` usable by `upload_image`.
- For all other actions: Text confirmation.

**Key behaviors:**
- Click coordinates use viewport origin (0,0 = top-left)
- `zoom` captures a specific rectangular region for closer inspection of small UI elements
- Screenshot `imageId` can be passed directly to `upload_image`

---

### 6. `navigate`

**Description:** Navigate to a URL or go forward/back in browser history.

**Input Schema:**
```typescript
{
  url: string;    // URL to navigate to (defaults to https:// if no protocol).
                  // Use "forward" or "back" for history navigation.
  tabId: number;  // Tab ID to navigate.
}
// required: ["url", "tabId"]
```

**Output:** Text confirmation of navigation.

---

### 7. `resize_window`

**Description:** Resize the current browser window.

**Input Schema:**
```typescript
{
  width: number;   // Target window width in pixels.
  height: number;  // Target window height in pixels.
  tabId: number;   // Tab ID identifying the window to resize.
}
// required: ["width", "height", "tabId"]
```

**Output:** Text confirmation.

---

### 8. `gif_creator`

**Description:** Manage GIF recording and export for browser automation sessions.

**Input Schema:**
```typescript
{
  action: "start_recording" | "stop_recording" | "export" | "clear";
    // "start_recording": begin capturing frames
    // "stop_recording": stop capturing but keep frames in memory
    // "export": generate and export GIF
    // "clear": discard all frames
  tabId: number;       // Tab ID identifying the tab group this applies to.

  download?: boolean;  // Set to true for "export" to download GIF in browser.
  filename?: string;   // Optional filename for exported GIF.
                       // Default: "recording-[timestamp].gif". For "export" only.
  options?: {
    showClickIndicators?: boolean; // Orange circles at click locations. Default: true.
    showDragPaths?: boolean;       // Red arrows for drag actions. Default: true.
    showActionLabels?: boolean;    // Black labels describing actions. Default: true.
    showProgressBar?: boolean;     // Orange progress bar at bottom. Default: true.
    showWatermark?: boolean;       // Claude logo watermark. Default: true.
    quality?: number;              // GIF compression quality 1–30.
                                   // Lower = better quality, slower. Default: 10.
  };
}
// required: ["action", "tabId"]
```

**Output:** Text confirmation of action performed.

**Key behaviors:**
- All operations are scoped to the tab's **group**, not just one tab
- Recommended: take a `computer` screenshot immediately after `start_recording` (captures initial state as first frame)
- Recommended: take a `computer` screenshot immediately before `stop_recording` (captures final state as last frame)
- For `export`: either provide `coordinate` to drag/drop to a page element, or set `download: true` to trigger browser download

---

### 9. `upload_image`

**Description:** Upload a previously captured screenshot or user-uploaded image to a file input or drag-and-drop target.

**Input Schema:**
```typescript
{
  imageId: string;         // ID of a previously captured screenshot (from
                           // computer tool's screenshot action) or user-uploaded image.
  tabId: number;           // Tab ID where the target element is located.
  ref?: string;            // Element reference ID from read_page/find. Use for
                           // file inputs (especially hidden). Provide either ref
                           // or coordinate, not both.
  coordinate?: number[];   // Viewport coordinates [x, y] for drag-and-drop
                           // to a visible location (e.g., Google Docs).
                           // Provide either ref or coordinate, not both.
  filename?: string;       // Optional filename for the uploaded file.
                           // Default: "image.png".
}
// required: ["imageId", "tabId"]
```

**Output:** Text confirmation.

**Key behaviors:**
- Two mutually exclusive upload approaches: (1) `ref` targets a specific or hidden file input; (2) `coordinate` performs a drag-and-drop to a visible location

---

### 10. `get_page_text`

**Description:** Extract raw text content from the page, prioritizing article content.

**Input Schema:**
```typescript
{
  tabId: number;  // Tab ID to extract text from.
}
// required: ["tabId"]
```

**Output:** Plain text without HTML formatting. Uses Readability-style article content prioritization.

---

### 11. `tabs_context_mcp`

**Title:** "Tabs Context"

**Description:** Get context information about the current MCP tab group.

**Input Schema:**
```typescript
{
  createIfEmpty?: boolean;  // Creates a new MCP tab group if none exists
                            // (a new Window with a new tab group containing
                            // an empty tab). No effect if group already exists.
}
// required: []
```

**Output:** JSON with `availableTabs` array:
```json
{
  "availableTabs": [
    { "tabId": 1, "title": "Page Title", "url": "https://example.com" }
  ]
}
```

**Key behaviors:**
- **CRITICAL:** Must be called at least once before any other browser automation tool
- Each new conversation should create its own new tab via `tabs_create_mcp` rather than reusing existing tabs, unless the user explicitly requests otherwise
- In a pool (multi-socket) scenario, calls all connected sockets and merges results (see Tab Routing below)

---

### 12. `tabs_create_mcp`

**Title:** "Tabs Create"

**Description:** Creates a new empty tab in the MCP tab group.

**Input Schema:**
```typescript
{}  // No properties.
// required: []
```

**Output:** Text confirmation with new tab ID.

**Key behaviors:**
- `tabs_context_mcp` must be called at least once before using this tool

---

### 13. `update_plan`

**Description:** Present a plan to the user for approval before taking browser automation actions. When approved, the listed domains are whitelisted for the session.

**Input Schema:**
```typescript
{
  domains: string[];   // List of domains to visit (e.g., ["github.com"]).
                       // These are approved for the session when the user
                       // accepts the plan.
  approach: string[];  // High-level steps: what you will do. Focus on outcomes
                       // and key actions. 3–7 items, be concise.
}
// required: ["domains", "approach"]
```

**Output:** Text confirmation: `"Plan updated"`.

**Domain approval flow (full lifecycle):**
1. AI calls `update_plan` with `domains` and `approach`
2. Tool is forwarded to Chrome extension, which presents the plan to the user in the browser
3. When user approves, extension responds with a `tool_result` indicating approval
4. Claude Code calls the internal `set_permission_mode` tool (handled by `fJK()` at line 30724) with `mode: "follow_a_plan"` and `allowed_domains: [...]`
5. `fJK()` validates mode is one of `["ask", "skip_all_permission_checks", "follow_a_plan"]`, then calls `client.setPermissionMode(mode, allowed_domains)`
6. On the bridge client (`$71`): `this.permissionMode = mode; this.allowedDomains = domains`
7. All subsequent bridge tool calls include `permission_mode: "follow_a_plan"` and `allowed_domains: [...]` in the wire message
8. Chrome extension skips per-action confirmation prompts for approved domains

**Key behaviors:**
- `set_permission_mode` is an internal tool not present in the `ip` array; it is handled entirely within the MCP server by `fJK()`
- UI renders the result as `"Plan updated"` in dim text (minimal display)
- Valid permission modes: `"ask"` (default), `"skip_all_permission_checks"`, `"follow_a_plan"`

---

### 14. `read_console_messages`

**Description:** Read browser console messages from a specific tab.

**Input Schema:**
```typescript
{
  tabId: number;       // Tab ID to read console from. (required)
  onlyErrors?: boolean; // If true, return only error and exception messages.
                        // Default: false.
  clear?: boolean;      // If true, clear messages after reading to avoid
                        // duplicates on subsequent calls. Default: false.
  pattern?: string;     // Regex pattern to filter messages
                        // (e.g., "error|warning", "MyApp").
                        // Always recommended to avoid irrelevant messages.
  limit?: number;       // Maximum messages to return. Default: 100.
}
// required: ["tabId"]
```

**Output:** Matching console messages. Automatically scoped to the current domain.

**Key behaviors:**
- Always recommended to provide a `pattern` to limit noise
- Messages from other domains are excluded automatically

---

### 15. `read_network_requests`

**Description:** Read HTTP network requests from a specific tab.

**Input Schema:**
```typescript
{
  tabId: number;        // Tab ID to read network requests from. (required)
  urlPattern?: string;  // Optional URL substring filter
                        // (e.g., "/api/", "example.com").
  clear?: boolean;      // If true, clear requests after reading. Default: false.
  limit?: number;       // Maximum requests to return. Default: 100.
}
// required: ["tabId"]
```

**Output:** Network requests including XHR, Fetch, documents, images, and cross-origin requests.

**Key behaviors:**
- Requests are automatically cleared when the page navigates to a different domain

---

### 16. `shortcuts_list`

**Description:** List all available shortcuts and workflows.

**Input Schema:**
```typescript
{
  tabId: number;  // Tab ID to list shortcuts from.
}
// required: ["tabId"]
```

**Output:** List of shortcuts with their commands, descriptions, and whether they are workflows. Shortcuts and workflows are interchangeable terms.

---

### 17. `shortcuts_execute`

**Description:** Execute a shortcut or workflow by running it in a new sidepanel window.

**Input Schema:**
```typescript
{
  tabId: number;          // Tab ID to execute the shortcut on. (required)
  shortcutId?: string;    // The ID of the shortcut to execute.
  command?: string;       // The command name (e.g., "debug", "summarize").
                          // Do not include a leading slash.
}
// required: ["tabId"]
```

**Output:** Text confirmation of execution start.

**Key behaviors:**
- Starts execution and returns immediately — does not wait for completion
- Runs in a new sidepanel window using the current tab

---

### 18. `switch_browser`

**Description:** Switch which Chrome browser is used for browser automation.

**Input Schema:**
```typescript
{}  // No properties.
// required: []
```

**Output:**
- Success: `Connected to browser "{name}".`
- No other browsers available: error explaining no other browsers available
- Timeout: error explaining no browser responded

**Key behaviors:**
- **BRIDGE-ONLY:** Available only with bridge connections (WebSocket to `bridge.claudeusercontent.com`). Returns an error on socket connections.
- Broadcasts a `pairing_request` to all Chrome browsers with the extension installed; user clicks "Connect" in the desired browser
- 120-second timeout for response
- Internally calls `q.switchBrowser?.()` which sets `selectedDeviceId = undefined`, `discoveryComplete = false`, then rebroadcasts
- `switch_browser` is excluded from the tool list (`ip.filter(t => t.name !== "switch_browser")`) on non-bridge (socket) connections

---

### Browser Tool Architecture

#### Variable `ip` and Lazy Initialization (`Al1`)

`ip` is a module-level variable assigned inside the `Al1` lazy-initializer:

```javascript
var Al1 = k(() => {
  ip = [ /* all 18 tool definition objects */ ];
});
```

`Al1` is a `k()`-wrapped lazy thunk — it executes only once; subsequent calls are no-ops. It is called by `r$A` (main browser module initializer, line 30858) and by the export block (line 30874).

#### `gi1()` — Client Factory (line 30820)

```javascript
function gi1(A) {
  return A.bridgeConfig ? O71(A) : A.getSocketPaths ? c$A(A) : w71(A);
}
```

Three client types:

| Client | Class | Transport | When Used |
|---|---|---|---|
| Bridge client | `$71` (via `O71`) | WebSocket to `bridge.claudeusercontent.com` | `bridgeConfig` present |
| Socket pool client | `d$A` (via `c$A`) | Pool of Unix domain sockets | `getSocketPaths` function provided |
| Single socket client | `IYA` (via `w71`) | Single Unix domain socket | Default fallback |

#### `$41()` — MCP Server Factory (`createClaudeForChromeMcpServer`, line 30822)

Creates an MCP server (`RC6`) with two request handlers:

1. **Tool list handler** (`CS6`): If `isDisabled?.()` returns true → `{ tools: [] }`. Otherwise:
   - Bridge connection: Returns all 18 tools in `ip` (including `switch_browser`)
   - Non-bridge connection: Returns `ip.filter(t => t.name !== "switch_browser")` (17 tools)

2. **Tool call handler** (`s46`): Dispatches via `i$A()`

#### `i$A()` — Tool Dispatcher (line 30786)

```javascript
var i$A = async (A, q, K, Y, z) => {
  if (K === "set_permission_mode") return fJK(q, Y);  // internal — handled locally
  if (K === "switch_browser")     return TJK(A, q);   // handled locally
  // All other tools forwarded to connected socket/bridge:
  let w = await q.ensureConnected();
  if (w) return await GJK(A, q, K, Y, z);
  return mi1(A);  // disconnected error message
};
```

`set_permission_mode` and `switch_browser` are handled entirely within the MCP server process. All 16 remaining tools are forwarded through the socket or bridge connection to the Chrome extension.

#### `GJK()` — Result Normalizer (line 30674)

Receives `{ result, error }` from socket/bridge. Key transformations:
- Content is `null`/`undefined` → returns generic `"Tool execution completed"` text
- Error with `"re-authenticated"` in text → triggers `onAuthenticationError()`
- Image handling: Content blocks with `type: "image"` and `source.data` are transformed to `{ type: "image", data: source.data, mimeType: source.media_type || "image/png" }`
- Returns `{ content: [...], isError: boolean }`

#### Tab Routing System

The pool client (`d$A`) manages multiple socket connections and routes tool calls by `tabId`:

```javascript
async callTool(A, q, K) {
  if (A === "tabs_context_mcp") return this.callTabsContext(q);
  let Y = q.tabId;
  if (Y !== undefined) {
    let w = this.tabRoutes.get(Y);  // look up which socket owns this tab
    if (w) {
      let c = this.clients.get(w);
      if (c?.isConnected()) return c.callTool(A, q);
    }
  }
  // fallback: first connected client
  let z = this.getConnectedClients();
  return z[0].callTool(A, q);
}
```

`tabRoutes`: `Map<tabId, socketPath>` — populated by `updateTabRoutes()` after each `tabs_context_mcp` call.

**Multi-socket `tabs_context_mcp` aggregation:** When called with multiple connected sockets, calls `tabs_context_mcp` on all sockets simultaneously via `Promise.allSettled`, stores the tab→socket mapping in `tabRoutes`, and merges all tabs into a single `{ availableTabs: [...] }` response.

#### Bridge Connection Details (`$71` class)

**WebSocket endpoints:**

| Environment | URL |
|---|---|
| Production | `wss://bridge.claudeusercontent.com` |
| Staging (`USE_STAGING_OAUTH`) | `wss://bridge-staging.claudeusercontent.com` |
| Local dev (`USE_LOCAL_OAUTH` or `LOCAL_BRIDGE`) | `ws://localhost:8765` |

**Authentication:** OAuth token (`getOAuthToken()`) and user UUID (`getUserId()`), sent in WebSocket handshake.

**Tool call wire format (sent to Chrome extension):**
```json
{
  "type": "tool_call",
  "tool_use_id": "<uuid>",
  "client_type": "claude-code",
  "tool": "<tool_name>",
  "args": { },
  "target_device_id": "<optional: specific Chrome extension device>",
  "permission_mode": "<optional: ask|skip_all_permission_checks|follow_a_plan>",
  "allowed_domains": ["domain1", "domain2"],
  "handle_permission_prompts": true
}
```

**Timeouts:**

| Scenario | Timeout |
|---|---|
| `tabs_context_mcp` | 2,000 ms (2 seconds) — collects from all connected extensions |
| All other tools | 120,000 ms (2 minutes) |
| Connection establishment | 10,000 ms (10 seconds) |
| `switch_browser` user response | 120,000 ms (2 minutes) |

**Device discovery:** On first tool call, queries connected Chrome extensions via `{ type: "list_extensions" }`. If only one extension found, auto-selects it. If multiple, broadcasts `pairing_request` for user to click "Connect" in desired browser.

**Incoming message types from extension:**
- `tool_result` — resolves the pending call promise
- `permission_request` — fires `onPermissionRequest` callback (if provided), then sends `permission_response`
- `notification` — forwarded as MCP notification
- `error` — clears `selectedDeviceId` so next call re-discovers
- `ping`/`pong` — keep-alive
- `extensions_list` — response to `list_extensions`, resolves discovery promise
- `peer_connected` — notifies that a new Chrome extension connected
- `pairing_accepted` — signals user clicked Connect in a browser

**Response normalization (`normalizeBridgeResponse()`):**
```
if (A.result || A.error) → return A as-is
if (A.content) {
  if (A.is_error) → { error: { content: A.content } }
  else            → { result: { content: A.content } }
}
```

#### Socket Client (`IYA` / `w71`)

**Transport:** Unix domain socket.

**Wire format:**
```json
{
  "method": "execute_tool",
  "params": {
    "client_id": "claude-code",
    "tool": "<tool_name>",
    "args": { }
  }
}
```

**Security validation (non-Windows):** Validates socket file permissions (must be `0600`, owned by current user) and socket directory permissions (must be `0700`, owned by current user).

**`setPermissionMode`:** No-op on socket client — permission modes are not forwarded through the native socket protocol.

**Retry:** Auto-reconnects and retries once on `SocketConnectionError`.

#### Conditional Loading

- `tengu_copper_bridge` feature flag disabled → bridge URL returns `undefined` (no bridge)
- `isDisabled?.()` check on the MCP server allows returning an empty tool list without stopping the server
- `switch_browser` is available **only** when `bridgeConfig` is present; all other 17 tools are available on both bridge and socket connections

#### UI Rendering (TUI)

The `byz()` function (line 523655) renders tool use messages in the terminal:

| Tool | TUI display |
|---|---|
| `computer` | action name + coordinate or ref |
| `navigate` | hostname of URL |
| `find` | truncated query |
| `resize_window` | `"WxH"` |
| `read_console_messages` | pattern + `"errors only"` flag |
| `read_network_requests` | URL pattern |
| `shortcuts_execute` | shortcutId |
| `javascript_tool` | full script text (verbose mode); empty string otherwise |
| `tabs_context_mcp`, `tabs_create_mcp`, `form_input`, `shortcuts_list`, `read_page`, `upload_image`, `get_page_text`, `update_plan` | empty string (no args displayed) |

A tab view link is shown for tools with a numeric `tabId` parameter, linking to `https://clau.de/chrome/tab/{tabId}`.

Result messages (`jSq()` function) show short status strings after completion: `"Navigation completed"`, `"Script executed"`, `"Page read"`, `"Plan updated"`, etc.

---

## 12. Appendix

### A. Numeric Constants Reference

All numeric constants found across tool implementations in cli.js:

| Constant / Expression | Value | Context |
|---|---|---|
| `tOq` | `50` | CronCreate: maximum simultaneous scheduled jobs |
| `recurringCapMs` | `900000` (15 min) | CronCreate: jitter cap for recurring tasks |
| `oneShotMaxMs` | `90000` (90 sec) | CronCreate: maximum early-fire jitter for one-shot tasks |
| `oneShotMinuteMod` | `30` | CronCreate: jitter applies only to tasks on `:00` or `:30` minute marks |
| Max cron search window | `527040` minutes (366 × 24 × 60) | CronCreate/`wk6()`: forward search limit for next fire time |
| Auto-expiry | 3 days | CronCreate: recurring tasks expire 3 days after creation |
| Feature flag TTL | `300000` ms (5 min) | `lC()`: Statsig gate cache TTL for `tengu_kairos_cron` |
| `maxResultSizeChars` | `100000` (100 KB) | CronCreate, CronDelete, CronList, SendMessage: max tool result size |
| `tabs_context_mcp` timeout | `2000` ms | Bridge client: collection window for multi-extension aggregation |
| Default tool timeout | `120000` ms (2 min) | Bridge client: per-tool-call timeout |
| Connection establishment timeout | `10000` ms (10 sec) | Bridge client: WebSocket connect timeout |
| `switch_browser` user timeout | `120000` ms (2 min) | Bridge client: wait for user to click Connect |
| `read_page` default `max_chars` | `50000` | `read_page`: output character limit |
| `read_page` default `depth` | `15` | `read_page`: default accessibility tree depth |
| `find` max results | `20` | `find`: maximum elements returned |
| `computer` `duration` max | `30` seconds | `computer` `wait` action |
| `computer` `scroll_amount` range | `1–10`, default `3` | `computer` `scroll` action |
| `computer` `repeat` range | `1–100`, default `1` | `computer` `key` action |
| `read_console_messages` default `limit` | `100` | `read_console_messages` |
| `read_network_requests` default `limit` | `100` | `read_network_requests` |
| `gif_creator` quality range | `1–30`, default `10` | `gif_creator`: lower = better quality |
| Bridge production URL | `wss://bridge.claudeusercontent.com` | Bridge client |
| Bridge staging URL | `wss://bridge-staging.claudeusercontent.com` | Bridge client (`USE_STAGING_OAUTH`) |
| Bridge local URL | `ws://localhost:8765` | Bridge client (`USE_LOCAL_OAUTH` or `LOCAL_BRIDGE`) |
| Socket file permissions | `0600` | Socket client: security validation (non-Windows) |
| Socket directory permissions | `0700` | Socket client: security validation (non-Windows) |
| GIF filename default | `"recording-[timestamp].gif"` | `gif_creator` |
| `upload_image` filename default | `"image.png"` | `upload_image` |
| Tab view link template | `https://clau.de/chrome/tab/{tabId}` | TUI rendering |
| Shutdown request ID format | `"shutdown-{Date.now()}@{recipient}"` | `SendMessage` `shutdown_request` |
| Team file path | `~/.claude/teams/{team-name}.json` | `SendMessage` `broadcast` |
| Durable tasks file | `.claude/scheduled_tasks.json` | CronCreate/CronDelete/`_k6()` |

---

### B. Tool Aliases (`aUA` map)

The `aUA` object maps legacy/alias tool names to their canonical internal names. Used by `bf(name)` (= `aUA[name] ?? name`) for resolution and by `sUA(canonical)` for reverse lookup.

```javascript
aUA = {
  Task:            Tq,   // "Task"            → "Agent"
  KillShell:       RI,   // "KillShell"       → "TaskStop"
  AgentOutputTool: SI,   // "AgentOutputTool" → "TaskOutput"
  BashOutputTool:  SI,   // "BashOutputTool"  → "TaskOutput"
  // ...{} (empty spread — extensible)
}
```

**MCP tool naming convention:** `K68(serverName, toolName)` → `"mcp__{serverName}__{toolName}"` with non-alphanumeric characters replaced by `_`. Parsed back by `ok(name)`: splits on `__`, validates `mcp` prefix.

Alias resolution is used in:
- `Sj(ruleString)` — permission rule parser
- `Jk6(tools, context)` — blocked-tool filter
- `Fi(context, tools)` — tool resolution function

---

### C. Variable Name Glossary

All tool name constants and key variable identifiers found in cli.js for Claude Code v2.1.71:

#### Tool Name Constants

| Variable | Value (string) | Tool |
|---|---|---|
| `Tq` | `"Agent"` | Agent (sub-task spawner) |
| `RI` | `"TaskStop"` | TaskStop (kill running sub-task) |
| `SI` | `"TaskOutput"` | TaskOutput (read sub-task output) |
| `p16` | `"EnterPlanMode"` | EnterPlanMode |
| `b_` | `"AskUserQuestion"` | AskUserQuestion |
| `f4` | `"Bash"` | Bash |
| `u4` | `"Read"` | Read |
| `VM` | `"WebFetch"` | WebFetch |
| `Yq` | `"Edit"` | Edit |
| `Y3` | `"Write"` | Write |
| `zz` | `"Glob"` | Glob |
| `fY` | `"Grep"` | Grep |
| `NM` | `"NotebookEdit"` | NotebookEdit |
| `tV` | `"WebSearch"` | WebSearch |
| `HF` | `"TodoWrite"` | TodoWrite |
| `nj` | `"Skill"` | Skill |
| `KX` | `"StructuredOutput"` | StructuredOutput |
| `OW` | `"ToolSearch"` | ToolSearch |
| `FT1` | `"EnterWorktree"` | EnterWorktree |
| `US` | `"TaskCreate"` | TaskCreate |
| `c16` | `"TaskGet"` | TaskGet |
| `l16` | `"TaskList"` | TaskList |
| `KL` | `"TaskUpdate"` | TaskUpdate |
| `Gx` | `"SendMessage"` | SendMessage |
| `HU` | `"CronCreate"` | CronCreate |
| `H_6` | `"CronDelete"` | CronDelete |
| `RS1` | `"CronList"` | CronList |

#### Other Key Constants

| Variable | Value | Context |
|---|---|---|
| `Aw` | `"team-lead"` | Team role constant |
| `nN` | `"claude-swarm"` | Team/swarm identifier constant |
| `aM` | (unresolved internal) | Plan-mode special tool (in `cT6` but kept in plan mode by `Ah8()`) |
| `BP3` | `null` | Platform-specific Bash variant (null → excluded from `dd`) |

---

### D. Tool Category Arrays

#### `CjY` — Write-Tracking Tools

```javascript
CjY = ["Edit", "Write", "NotebookEdit"]
```

Used by `pC8(name)` to identify which tool calls trigger write-decision telemetry. `dC8()` extracts file path and language for metrics from these tool calls.

#### `Y68.filePatternTools` — File Pattern Permission Tools

```javascript
filePatternTools: ["Read", "Write", "Edit", "Glob", "NotebookRead", "NotebookEdit"]
```

Used by `tUA(name)`. These tools support file pattern matching in `allowedTools` permission rules (e.g., `allowedTools("Edit(src/**)")`).

#### `Y68.bashPrefixTools` — Bash Prefix Permission Tools

```javascript
bashPrefixTools: ["Bash"]
```

Used by `eUA(name)`. Bash supports prefix-based permission rules (e.g., `allowedTools("Bash(git *)")`).

#### `Y68.customValidation` — Per-Tool Validation Functions

```javascript
customValidation: {
  WebSearch: (A) => {
    if (A.includes("*") || A.includes("?"))
      return { valid: false, error: "WebSearch does not support wildcards", ... };
    return { valid: true };
  }
}
```

#### `gP3` — Safe Read-Only Tools

```javascript
gP3 = [...dd, zz, fY, u4, VM, tV]
// Expands to: ["Bash", "Glob", "Grep", "Read", "WebFetch", "WebSearch"]
```

Used for permission prompting category: these tools are classified as safe/read-only for user-facing permission decisions.

#### `FP3` — Write Tools (Permission Category)

```javascript
FP3 = [Yq, Y3, NM]
// Expands to: ["Edit", "Write", "NotebookEdit"]
```

Used for permission prompting category: these tools require write-permission verification.

#### `dd` — Platform Bash Tools

```javascript
dd = [f4, BP3].filter((A) => A != null)
// f4 = "Bash", BP3 = null
// Result: dd = ["Bash"]
```

Platform-specific Bash tool array. `BP3` is a platform-conditional Bash variant (null on non-macOS or unsupported platforms).

#### `wL_` — Search/Read Tools Set

```javascript
wL_ = new Set([u4, ...dd, fY, zz, tV, VM, Yq, Y3, /* ...[] */])
// = new Set(["Read", "Bash", "Grep", "Glob", "WebSearch", "WebFetch", "Edit", "Write"])
```

#### `QT1` — Async-Safe Tools

Tools allowed during async/background operations (`isAsync: true` mode). When an Agent sub-task runs asynchronously, only these tools are available:

```javascript
QT1 = new Set([
  u4,    // "Read"
  tV,    // "WebSearch"
  HF,    // "TodoWrite"
  fY,    // "Grep"
  VM,    // "WebFetch"
  zz,    // "Glob"
  ...dd, // "Bash"
  Yq,    // "Edit"
  Y3,    // "Write"
  NM,    // "NotebookEdit"
  nj,    // "Skill"
  KX,    // "StructuredOutput"
  OW,    // "ToolSearch"
  FT1,   // "EnterWorktree"
])
```

#### `Gf4` — Team Async-Safe Tools

Additional tools allowed in async mode when team features are active (`Z7() && AW()`):

```javascript
Gf4 = new Set([
  US,   // "TaskCreate"
  c16,  // "TaskGet"
  l16,  // "TaskList"
  KL,   // "TaskUpdate"
  Gx,   // "SendMessage"
])
```

#### `cT6` — Always-Hidden Tools

Tools permanently filtered out of all tool lists shown to the model:

```javascript
cT6 = new Set([SI, aM, p16, Tq, b_, RI])
// = new Set(["TaskOutput", <internal-plan-tool>, "EnterPlanMode", "Agent", "AskUserQuestion", "TaskStop"])
```

**Exception:** `aM` (internal plan-mode tool) is retained in plan mode: `Ah8()` keeps it when `R5(z, aM) && permissionMode === "plan"`.

`eC8 = new Set([...cT6])` — same set used specifically for filtering non-built-in (external) tool sources.

#### `BROWSER_TOOLS` Export

```javascript
BROWSER_TOOLS: () => ip  // line 30870
```

Lazy accessor returning the full array of 18 browser tool definitions.

---

### E. Tool List Assembly (`YU()` and `pP()`)

**`YU()` — Full tool list** (simplified, line ~451550):
```javascript
function YU() {
  return [
    /* core tools: dT6, XS1, Hq, HX, KY (Read), dP, gP, ln, UP, pN, PS1, MS1,
       ev6, jA6, Ka6, tc8, an, tn (names unresolved in research) */
    ...(cH() ? [] : [zU, bu]),        // excluded in cH() mode
    ...(iH() ? [$Oq, WOq, EOq, uOq] : []),  // included in simple/headless mode
    ...(FHq ? [FHq] : []),
    ...(QHq ? [QHq] : []),
    ...(Yk6() ? [F$q] : []),
    ...(Z7() ? [vzz(), kzz(), Ezz()] : []),  // TeamCreate, TeamDelete, SendMessage
    ...(gHq ? [gHq] : []),
    ...(uHq ? [uHq] : []),
    ...Nzz,  // CronCreateTool, CronDeleteTool, CronListTool
    ...(BHq ? [BHq] : []),
    ...(mHq ? [mHq] : []),
    ...(pHq?.() ? [pHq()] : []),
    ...(UHq ? [UHq] : []),
    ...(CS1 ? [CS1] : []),
    ...(hS1 ? [hS1] : []),
    ...(IS1 ? [IS1] : []),
    ...(bS1 ? [bS1] : []),
    ...(Wx() ? [wd6] : []),
  ];
}
```

**`pP(A)` — Effective built-in tool list:**
- In `CLAUDE_CODE_SIMPLE` mode: only `[Hq, KY, dP]` with blocked-tool filtering
- Otherwise: filters `YU()` by `cT6` exclusion set, blocked rules, and `isEnabled()` on each tool
- In `CLAUDE_REPL_MODE`: additionally filters tools overlapping with `pV1` set

**`HA6(A, q)` — Final tool list (built-in + external MCP):**
```javascript
function HA6(A, q) {
  let K = pP(A);           // enabled built-in tools
  if ($1(process.env.CLAUDE_CODE_SIMPLE)) return K;
  let Y = Jk6(q, A);       // filtered external/MCP tools
  return zW([...K, ...Y], "name");  // deduplication by name
}
```
