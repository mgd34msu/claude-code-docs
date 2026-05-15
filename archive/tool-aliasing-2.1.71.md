# Claude Code v2.1.71 — Tool Aliasing & Tool Dispatch Deep-Dive

**Source**: `cli.js` (612,918 lines, minified bundle)  
**Feature gate**: `tengu_tool_input_aliasing` (Statsig, default: `false`)  
**Date**: 2026-03-09

---

## Overview

Two independent aliasing systems exist inside Claude Code's tool dispatch pipeline:

1. **Parameter aliasing** (`inputParamAliases`) — remaps input parameter names before schema validation. Claude sends `{ filePath: "foo.ts" }` and the tool receives `{ file_path: "foo.ts" }`. Currently gated behind `tengu_tool_input_aliasing` (default off).

2. **Tool name aliasing** (`aliases[]`) — maps alternative *tool* names to a tool definition so `"Write"` and `"write"` (or any custom alias) resolve to the same tool at dispatch time. Always active, no feature gate. Currently unused for native tool dispatch; only slash commands use it.

Beyond these two, a third mechanism — **tool shadowing** — emerges from the assembly order: extra/plugin tools are prepended before native tools, and the dedup strategy keeps the first occurrence. Any injected tool with the same name as a native tool silently replaces it.

---

## Table of Contents

1. [Parameter Aliasing (inputParamAliases)](#1-parameter-aliasing-inputparamaliases)
   - 1.1 The Aliasing Function (`gL1`)
   - 1.2 Where Aliasing is Applied
   - 1.3 Tools with Parameter Aliases
   - 1.4 Alias Data Structure
2. [Tool Name Aliasing (aliases[])](#2-tool-name-aliasing-aliases)
   - 2.1 The `aliases` Field
   - 2.2 The `R5()` Resolver
   - 2.3 Current Usage
3. [Tool Dispatch Pipeline](#3-tool-dispatch-pipeline)
   - 3.1 Tool Registration
   - 3.2 Name Resolution Order
   - 3.3 Tool Execution Flow
4. [Tool Shadowing](#4-tool-shadowing)
   - 4.1 How Shadowing Works
   - 4.2 SDK Tool Shadowing
   - 4.3 Plugin Tool Shadowing
   - 4.4 Practical Implications
5. [MCP Tool Integration](#5-mcp-tool-integration)
   - 5.1 How MCP Tools are Registered
   - 5.2 MCP Tool Naming
   - 5.3 MCP Tools Cannot Define Parameter Aliases
6. [Answering the Key Question](#6-answering-the-key-question)
7. [Key Code Locations](#7-key-code-locations)
8. [Telemetry](#8-telemetry)

---

## 1. Parameter Aliasing (inputParamAliases)

### 1.1 The Aliasing Function (`gL1`)

**Location**: line 452053

```javascript
function gL1(A, q) {
  // A = tool definition object
  // q = raw input parameters object from the model
  if (!A.inputParamAliases || !p8("tengu_tool_input_aliasing", false)) return q;

  let K = A.inputParamAliases,  // alias map: { aliasName: canonicalName }
    Y = {},                      // output params (new object)
    z = [];                      // log of applied aliases

  for (let [w, _] of Object.entries(q)) {
    let $ = K[w];           // look up this param name in alias map
    if ($ && !($ in q)) {
      // param matches an alias AND canonical param is not already present
      Y[$] = _;             // write value under canonical name
      z.push(`${w}->${$}`); // log the remapping
    } else {
      Y[w] = _;             // no alias or canonical already present — pass through
    }
  }

  if (z.length > 0)
    return (
      c("tengu_tool_input_alias_applied", {  // fire analytics event
        toolName: wK(A.name),
        aliases: z.join(","),
      }),
      Y  // return remapped params
    );
  return q;  // no aliases applied — return original
}
```

**Key semantics:**

- **Gate first**: if `tengu_tool_input_aliasing` is `false` (the default), `gL1` returns `q` unchanged. The entire function is a no-op in production today.
- **Alias wins only if canonical is absent**: the condition `$ && !($ in q)` means if Claude sends both `filePath` and `file_path`, the alias is ignored and the canonical value passes through unchanged. No accidental overrides.
- **Non-destructive**: returns a new object `Y`; does not mutate the original `q`.
- **No alias chaining**: the map is flat — `{ aliasName: "canonicalName" }` — one level only. Alias-of-an-alias is not resolved.
- **Telemetry fires** only when at least one alias was actually applied.

### 1.2 Where Aliasing is Applied

Aliasing is injected at exactly **two points** in the execution pipeline, always before schema validation:

**Injection point 1 — `xeY()` (line 408884) — Sequential execution path**

```javascript
function xeY(A, q) {
  return A.reduce((K, Y) => {
    let z = z5(q.options.tools, Y.name);  // look up tool by name
    if (z) Y.input = gL1(z, Y.input);    // <<< alias remapping
    let w = z?.inputSchema.safeParse(Y.input),  // then validate (post-remap)
      _ = w?.success
        ? (() => Boolean(z?.isConcurrencySafe(w.data)))()
        : false;
    if (_ && K[K.length - 1]?.isConcurrencySafe) K[K.length - 1].blocks.push(Y);
    else K.push({ isConcurrencySafe: _, blocks: [Y] });
    return K;
  }, []);
}
```

This function groups tool calls into concurrency batches. Aliasing happens before grouping and before the `Ur6` dispatcher ever sees the input. By the time `Ur6` receives `A.input`, it is already canonicalized.

**Injection point 2 — `addTool()` (line 452872) — Streaming / scheduler path**

```javascript
addTool(A, q) {
  let K = z5(this.toolDefinitions, A.name);
  if (!K) { /* error: unknown tool */ return; }
  A.input = gL1(K, A.input);  // <<< alias remapping
  let Y = K.inputSchema.safeParse(A.input),
    z = Y?.success ? Boolean(K.isConcurrencySafe(Y.data)) : false;
  this.tools.push({ ... });
  this.processQueue();
}
```

The streaming scheduler path applies aliasing before queuing the tool call for execution. Same semantics, different code path.

**Important**: `Ur6()` (the dispatcher) does NOT call `gL1`. It receives `j = A.input` already remapped and passes it directly to `Szz` → `Czz` for schema validation and execution. Aliasing must happen upstream of validation — and it does.

### 1.3 Tools with Parameter Aliases

Five native tools currently define `inputParamAliases`:

| Tool | Variable | Name | Line | Aliases Defined |
|------|----------|------|------|-----------------|
| Read | `KY` | `u4` | 249777 | `filePath`, `filepath`, `path` → `file_path` |
| Write | `gP` | `Y3` | 426672, 452102 | `filePath`, `filepath`, `path` → `file_path` |
| Edit | `dP` | `Yq` | 525999 | `old_str`, `oldString` → `old_string`; `new_str`, `newString` → `new_string`; `filePath`, `filepath`, `path` → `file_path` |
| Grep | `bu` | — | 427215 | `c`, `C` → `-C`; `a`, `A` → `-A`; `b`, `B` → `-B`; `n` → `-n`; `i` → `-i`; `include` → `glob`; `regex`, `search` → `pattern`; `directory` → `path` |
| Glob/Find | `zU` | `zz` | 427553 | `directory` → `path` |

All five are core file I/O tools. The Grep tool has by far the most aliases — it accommodates common command-line flag shorthands (`-C`, `-A`, `-B`), semantic synonyms (`regex`, `search`), and structural synonyms (`directory`, `include`).

### 1.4 Alias Data Structure

```typescript
// Field on the tool definition object:
inputParamAliases?: Record<string, string>;
// key   = alias name (what Claude might send)
// value = canonical name (what the Zod schema expects)

// Example — Edit tool:
inputParamAliases: {
  old_str:    "old_string",   // snake_case short form
  new_str:    "new_string",
  oldString:  "old_string",   // camelCase variant
  newString:  "new_string",
  filePath:   "file_path",    // camelCase file path
  filepath:   "file_path",    // lowercase variant
  path:       "file_path",    // bare path
}
```

This is a plain JavaScript object — not part of the Zod schema. The Zod schema validates canonical parameter names only. Aliasing pre-processes inputs before Zod ever sees them.

---

## 2. Tool Name Aliasing (aliases[])

### 2.1 The `aliases` Field

A tool definition can declare an array of alternative names it responds to:

```typescript
interface ToolDefinition {
  name: string;        // canonical name (primary)
  aliases?: string[];  // alternative names this tool answers to
  // ...
}
```

Unlike `inputParamAliases`, this has no feature gate — it is always active.

### 2.2 The `R5()` Resolver

**Location**: line 89345

```javascript
function R5(A, q) {
  // Does tool A answer to name q?
  return A.name === q || (A.aliases?.includes(q) ?? false);
}

function z5(A, q) {
  // Find the first tool in array A that answers to name q
  return A.find((K) => R5(K, q));
}
```

`z5` is the universal tool lookup function used throughout the codebase. By checking `aliases` inside `R5`, any code that uses `z5` automatically respects name aliases without further changes.

**The dispatch fallback in `Ur6` (line 452100-452101)**:

```javascript
async function* Ur6(A, q, K, Y) {
  let z = A.name,
    w = z5(Y.options.tools, z);  // 1. Look in session-registered tools (primary + aliases)
  if (!w) {
    let J = z5(YU(), z);          // 2. Fall back to full native registry
    if (J && J.aliases?.includes(z)) w = J;  // 3. Only accept if matched via alias
  }
  // if still !w: error "No such tool available"
  // ...
}
```

The fallback in step 2-3 is narrow: it only accepts a native tool from `YU()` if the name was matched via an alias — not via the primary name. This prevents the fallback from accidentally picking up a native tool that was intentionally excluded from the session's registered tools.

### 2.3 Current Usage

- **Native tools**: none of the 5 aliased-parameter tools define an `aliases` array. The field is part of the tool interface but no current native tool uses it for dispatch.
- **Slash commands**: the `aliases` field IS used by slash command resolution (line 598953) to map `/command` shorthand to full command definitions — same `R5`/`z5` mechanism, different domain.
- **Plugin/custom tools**: any tool added to the registry can define `aliases`, and `z5` will find it automatically.

---

## 3. Tool Dispatch Pipeline

### 3.1 Tool Registration

Native tools are defined in the `YU()` function (line 451445):

```javascript
function YU() {
  return [
    dT6, XS1, Hq,
    ...(cH() ? [] : [zU, bu]),   // Glob + Grep excluded on certain platforms
    HX, KY, dP, gP, ln, UP, pN, PS1, MS1, ev6, jA6, Ka6,
    ...(iH() ? [$Oq, WOq, EOq, uOq] : []),  // team tools if enabled
    ...(FHq ? [FHq] : []),
    ...(QHq ? [QHq] : []),
    tc8,
    ...(Yk6() ? [F$q] : []),
    ...(Z7() ? [vzz(), kzz(), Ezz()] : []),  // additional team tools
    ...(gHq ? [gHq] : []),
    ...(uHq ? [uHq] : []),
    ...Nzz,   // injectable array — other code can push tools here
    ...(BHq ? [BHq] : []),
    ...(mHq ? [mHq] : []),
    ...(pHq?.() ? [pHq()] : []),
    ...(UHq ? [UHq] : []),
    an, tn,
    ...(CS1 ? [CS1] : []),
    ...(hS1 ? [hS1] : []),
    ...(IS1 ? [IS1] : []),
    ...(bS1 ? [bS1] : []),
    ...(Wx() ? [wd6] : []),
  ];
}
```

At session start (line 574782), the session tool list is assembled:

```javascript
let b6 = HA6(p6.toolPermissionContext, p6.mcp.tools);
// HA6 = zW([...pP(A), ...Jk6(q, A)], "name")
//         native tools     MCP tools
//       (native comes first in HA6)

let R6 = zW(
  CE6([...Y, ...V6, ...x, ...U.tools], b6, p6.toolPermissionContext.mode),
  "name",
);
// CE6(A, q) = zW([...A, ...q], "name")  — extra tools BEFORE HA6 result
// zW = uniqBy(first)                    — first occurrence wins
// R6 becomes Y.options.tools
```

### 3.2 Name Resolution Order

Full priority order (first-wins via `zW = uniqBy("name")`):

```
Priority 1 (highest): Y          — explicit extra tools passed in options
Priority 2:           V6         — SDK plugin tools (from deferred MCP, filtered)
Priority 3:           x          — SDK state tools
Priority 4:           U.tools    — dynamic SDK tools
Priority 5:           pP(A)      — native tools (filtered by permissions)
Priority 6 (lowest):  Jk6(q,A)  — standard MCP tools (filtered by permissions)
```

Within the `HA6` group, native tools shadow MCP tools. The entire `HA6` result is shadowed by anything in the extra tools array.

### 3.3 Tool Execution Flow

```
Claude produces tool_use block { name: "Write", input: { filePath: "foo.ts", content: "..." } }
          |
          v
    xeY() — sequential path, or addTool() — streaming path
          |
          |--- z5(options.tools, name) — look up tool definition
          |--- gL1(toolDef, input)     — remap aliases (if gate enabled)
          |         filePath -> file_path
          |--- inputSchema.safeParse() — Zod validation on canonicalized input
          |--- isConcurrencySafe()     — concurrency grouping decision
          |
          v
    Ur6(toolCall, ..., options)
          |
          |--- z5(options.tools, name) — resolve tool definition again
          |--- (fallback to YU() only if not found, and only via alias)
          |--- j = A.input             — already alias-remapped
          |
          v
    Szz(toolDef, id, j, options, ...)
          |
          v
    Czz() — runs PreToolUse hook
          |         hookUpdatedInput can replace j here
          |--- toolDef.call(j, context, ...)
          |
          v
    Tool executes
```

Aliasing happens **once**, early, in `xeY`/`addTool`. Everything downstream sees the canonical parameter names.

---

## 4. Tool Shadowing

### 4.1 How Shadowing Works

Shadowing is a consequence of `zW = uniqBy("name")` applied to a concatenated list where extra tools come first:

```javascript
// CE6 source:
function CE6(A, q) {
  return zW([...A, ...q], "name");
  // A = extra/plugin/SDK tools
  // q = native + MCP tools (HA6 result)
}
```

If any tool in `A` has `name: "Write"`, the native `Write` tool (`gP`) in `q` is dropped silently. The session's `options.tools` array contains the injected version, and `Ur6` dispatches to it.

**The native tool is not deleted** — it still lives in `YU()` and in `HA6`. But since `Ur6` checks `options.tools` first (step 1) and the injected tool is there, the native tool is never reached.

### 4.2 SDK Tool Shadowing

Normally MCP tools are named `mcp__<serverName>__<toolName>` and never collide with native tool names. There is one escape hatch:

```javascript
// Line ~525374 in MCP tool construction:
let Y = A.config.type === "sdk" &&
  $1(process.env.CLAUDE_AGENT_SDK_MCP_NO_PREFIX);
return {
  name: Y ? z.name : w,  // Y=true → raw tool name, no mcp__ prefix
  // ...
};
```

Conditions required for SDK MCP shadowing:
1. The MCP server must be of `type: "sdk"` (not a standard external server)
2. `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` environment variable must be set to a truthy value
3. The tool's raw name must match an existing native tool name (e.g., `"Write"`)

When all three are true, the SDK MCP tool is inserted into the extra tools array (`V6`) with priority 2 — above native tools, below explicit extra tools.

### 4.3 Plugin Tool Shadowing

Plugin tools inside `YU()` via `Nzz` (the injectable array) or conditional singletons (`CS1`, `hS1`, etc.) do NOT shadow native tools — they are all part of the same `pP(A)` call which goes through `HA6`. Within `HA6`, `zW` is applied: the native tools in `pP(A)` appear first in the spread `[...pP(A), ...Jk6(q, A)]`, so native tools shadow injected `YU()` additions.

**But**: plugin tools delivered via the extra tools arrays (`Y`, `V6`, `x`, `U.tools`) — i.e., tools passed to the session constructor, not registered in `YU()` — DO shadow native tools because they are prepended in `CE6`.

Summary:

| Injection Method | Can Shadow Native Tools? |
|-----------------|-------------------------|
| `Nzz` push (inside `YU()`) | No — native tools come first in `HA6` |
| Conditional singletons in `YU()` | No — same reason |
| Explicit extra tools (`Y` param) | Yes — priority 1 |
| SDK plugin tools (`V6`) | Yes — priority 2 |
| SDK state tools (`x`) | Yes — priority 3 |
| Dynamic SDK tools (`U.tools`) | Yes — priority 4 |
| MCP tool with `NO_PREFIX` | Yes — via `V6` path, priority 2 |

### 4.4 Practical Implications

The tool that executes is determined entirely by what is in `options.tools` at session time. There is no runtime permission check that says "this call came from a shadowed tool, abort". The shadow is transparent to the dispatch machinery.

A shadowing tool inherits none of the native tool's behavior — it must implement its own `call()`, `inputSchema`, `isConcurrencySafe()`, etc. If it does not, the call will fail or behave incorrectly. The shadow is a complete replacement.

---

## 5. MCP Tool Integration

### 5.1 How MCP Tools are Registered

MCP tools are constructed at runtime from server discovery. The constructor (line 525374) spreads a base template and adds server-provided fields:

```javascript
return {
  ...$Sq,                  // base MCP tool template
  name: Y ? z.name : w,   // mcp__serverName__toolName, or raw name if NO_PREFIX
  mcpInfo: { serverName: A.name, toolName: z.name },
  isMcp: true,
  async description() { return z.description ?? ""; },
  isConcurrencySafe() { return z.annotations?.readOnlyHint ?? false; },
  isReadOnly() { return z.annotations?.readOnlyHint ?? false; },
  inputJSONSchema: z.inputSchema,  // raw JSON schema — NOT Zod
  async call(_, $, O, H, j) { /* calls MCP server */ },
  // inputParamAliases is NOT here
};
```

There is no `inputParamAliases` field. The MCP protocol does not define a parameter aliasing concept, and the constructor does not synthesize one.

### 5.2 MCP Tool Naming

Default format: `mcp__<serverName>__<toolName>` — constructed by `K68()`. This guarantees no collision with native tool names (which are never prefixed this way).

Non-prefixed mode: only when `CLAUDE_AGENT_SDK_MCP_NO_PREFIX=1` AND server type is `"sdk"`. In that mode, raw tool names are used and shadowing is possible.

### 5.3 MCP Tools Cannot Define Parameter Aliases

Three reasons:

1. **Constructor omission**: `inputParamAliases` is not spread from `$Sq` or added from `z` (the server-provided tool definition).
2. **Schema type mismatch**: MCP tools use `inputJSONSchema` (raw JSON Schema object), while `gL1` operates on the `inputParamAliases` field of the tool definition object — a completely separate metadata field.
3. **No MCP protocol support**: MCP's tool schema spec has no aliasing concept. There is nowhere in the protocol for a server to declare parameter aliases.

If you need parameter aliasing for an MCP tool, the only path is to wrap it in a native-style tool definition injected as an extra tool.

---

## 6. Answering the Key Question

**Can a plugin or MCP server replace a native tool like `Write` with `precision_write`?**

### Yes — via Tool Shadowing (recommended path)

Inject a tool definition with `name: "Write"` as an extra tool at session construction time. Since extra tools (priority 1-4) are prepended before native tools, the injected `"Write"` wins in `zW` dedup and the native `gP` tool is silently dropped from `options.tools`.

```
Session construction:
  extras = [{ name: "Write", call: precisionWriteImpl, inputSchema: ... }]
  b6     = HA6(...)   ← contains native Write (gP)
  R6     = zW(CE6(extras, b6), "name")
            ↑ extras[0].name == "Write" wins
            ↑ native Write is deduplicated out

Claude calls Write:
  z5(options.tools, "Write") → finds your injected tool
  Ur6 dispatches to your tool
  Native gP never runs
```

Requirements:
- Your tool must implement the full `ToolDefinition` interface: `inputSchema` (Zod), `call()`, `isConcurrencySafe()`, `isReadOnly()`, `description()`, `userFacingName()`, etc.
- Your `inputSchema` should accept the same parameters as the native `Write` — or a superset — to avoid Claude producing inputs that fail validation.
- If you want parameter aliasing on your replacement tool, add `inputParamAliases` to your definition and enable `tengu_tool_input_aliasing`.

### Yes — via SDK MCP + NO_PREFIX (alternative path)

If operating in SDK mode:
1. Set `CLAUDE_AGENT_SDK_MCP_NO_PREFIX=1`
2. Define an MCP tool named `"Write"` on an SDK-type server
3. The tool enters the session as a priority-2 extra (via `V6`), shadowing native `Write`

Limitation: this tool will NOT support `inputParamAliases` (MCP tools cannot define them). If you want parameter remapping, you must handle it inside the `call()` implementation.

### No — via Parameter Aliasing alone

Parameter aliasing (`inputParamAliases`) only remaps **input parameter names**. It does not redirect tool dispatch. There is no way to make `gL1` route a call for `"Write"` to a different tool — `gL1` receives the tool definition as `A` after `z5` has already resolved it. You cannot alias `"Write"` to `"precision_write"` using `inputParamAliases`.

### No — via Tool Name Aliasing (`aliases[]`) alone

Adding `aliases: ["Write"]` to `precision_write` does not help if `"Write"` already exists in `options.tools`. The `Ur6` fallback (step 2) only fires when the tool is NOT found in `options.tools`. Since the native `Write` tool IS registered by default, `z5(options.tools, "Write")` finds it in step 1 and never reaches the fallback.

For `aliases` to work as a replacement, you would need to also remove the native `Write` from the session tool list — at which point you have effectively done tool shadowing anyway.

### Decision Matrix

| Goal | Mechanism | Works? | Gate Required? |
|------|-----------|--------|----------------|
| Remap `filePath` → `file_path` on Write | `inputParamAliases` | Yes | `tengu_tool_input_aliasing` |
| Make `"Write"` call your custom tool | Extra tool with `name: "Write"` | Yes | None |
| MCP tool shadows native Write | `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` + SDK type | Yes | None (env var) |
| Add alias `"write"` for Write | `aliases: ["write"]` on tool def | Partial — only if Write is unregistered | None |
| MCP tool defines param aliases | Not possible | No | N/A |
| `gL1` routes to a different tool | Not possible | No | N/A |

---

## 7. Key Code Locations

| Symbol / Function | Line | Purpose |
|-------------------|------|---------|
| `gL1` | 452053 | Core parameter aliasing function |
| `xeY` | 408884 | Sequential path — applies aliasing, groups concurrency batches |
| `addTool` | 452872 | Streaming path — applies aliasing before queuing |
| `R5` | 89345 | Tool name matcher — checks primary name and `aliases[]` |
| `z5` | 89348 | Tool lookup — finds first tool matching name via `R5` |
| `Ur6` | 452100 | Tool dispatcher — resolves tool, calls `Szz` |
| `Szz` / `Czz` | ~452360 | Schema validation + hook execution (PreToolUse) |
| `YU` | 451445 | Native tool registry — returns all native tool definitions |
| `HA6` | ~574760 | Merges native tools + MCP tools (native-first dedup) |
| `CE6` | ~574780 | Merges extra tools + native+MCP (extra-first dedup) |
| `zW` | (util) | `uniqBy("name")` — first-wins deduplication |
| `Nzz` | (module) | Injectable array for additional tools inside `YU()` |
| MCP tool constructor | 525374 | Builds MCP tool definition from server discovery |
| `K68` | (util) | Builds `mcp__server__tool` name string |
| Read tool (`KY`) | 249777 | `inputParamAliases` for `file_path` |
| Write tool (`gP`) | 426672, 452102 | `inputParamAliases` for `file_path` |
| Grep tool (`bu`) | 427215 | `inputParamAliases` — most extensive alias set |
| Glob tool (`zU`) | 427553 | `inputParamAliases` — `directory` → `path` |
| Edit tool (`dP`) | 525999 | `inputParamAliases` for `file_path`, `old_string`, `new_string` |

---

## 8. Telemetry

When parameter aliasing is active and at least one alias is applied, `gL1` fires:

```javascript
c("tengu_tool_input_alias_applied", {
  toolName: wK(A.name),  // sanitized/canonical tool name
  aliases: z.join(","),  // comma-separated list of applied remappings, e.g. "filePath->file_path"
});
```

- **Event name**: `tengu_tool_input_alias_applied`
- **Platform**: Statsig analytics (`c()` is the analytics event emitter)
- **`toolName`**: the tool's canonical name, passed through `wK()` for sanitization
- **`aliases`**: a comma-separated string of applied remappings in `aliasName->canonicalName` format
- **Fires only when**: at least one alias was actually remapped (i.e., `z.length > 0`)
- **Does not fire when**: gate is disabled, no aliases defined on tool, or no alias names matched in the input

This event is the only telemetry hook in the aliasing system. There is no separate event for parameter aliasing being skipped or for the gate being disabled.
