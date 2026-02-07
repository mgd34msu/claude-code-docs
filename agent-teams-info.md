# Claude Code CLI - Agent Teams Reference
> Info as of Claude Code CLI v2.1.34
> Research completed: 2026-02-06

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Directory Structure & File Paths](#2-directory-structure--file-paths)
3. [Team Creation & Deletion](#3-team-creation--deletion)
4. [Team Config File Schema](#4-team-config-file-schema)
5. [Agent Types & Spawning](#5-agent-types--spawning)
6. [Agent Lifecycle](#6-agent-lifecycle)
7. [Permission Modes](#7-permission-modes)
8. [Plan Mode](#8-plan-mode)
9. [Tool Restrictions](#9-tool-restrictions)
10. [Task Management System](#10-task-management-system)
11. [Inter-Agent Messaging](#11-inter-agent-messaging)
12. [UI & Status Rendering](#12-ui--status-rendering)
13. [System Prompts for Teams](#13-system-prompts-for-teams)
14. [Telemetry Events](#14-telemetry-events)
15. [Key Functions Reference Table](#15-key-functions-reference-table)
16. [Architecture Diagram](#16-architecture-diagram)

---

## 1. Overview & Architecture

### What Teams Are

The Claude Code CLI implements a sophisticated agent teams system that allows multiple AI agents to coordinate work together. Teams use a **file-based coordination model** where configuration, tasks, and messages are stored in JSON files in `~/.claude/`.

### Lead-Centric Model

- **One team lead** coordinates the team
- Team lead creates the team and spawns teammates
- Team lead can assign tasks, approve plans, and request shutdowns
- Teammates work on assigned tasks and communicate via messaging

### Key Characteristics

- **File-based coordination**: Teams use JSON files rather than in-memory state for persistence and cross-process communication
- **Lead-centric model**: One agent is designated as team lead, responsible for team lifecycle
- **Asynchronous messaging**: File-based message queue with polling-based delivery
- **Task-based workflow**: Shared task list with ownership and dependencies
- **Permission isolation**: Each agent can have different permission modes and tool restrictions

---

## 2. Directory Structure & File Paths

### Base Configuration Directory

**Location**: `~/.claude/` (or `$CLAUDE_CONFIG_DIR`)

```typescript
function $8() {
  return process.env.CLAUDE_CONFIG_DIR ?? FdA(bxq(), ".claude");
}
```

### Teams Directory

**Location**: `~/.claude/teams/`

```typescript
function MW() {
  return FdA($8(), "teams");
}
```

### Individual Team Directory Structure

For a team named "my-team":
```
~/.claude/teams/my-team/
  ├── config.json        # Team configuration and member registry
  └── inboxes/           # Message inboxes for team members
      ├── {agent-id}.json
      └── ...

~/.claude/tasks/my-team/  # Task list directory (separate from teams dir)
  ├── 1.json          # Task with ID "1"
  ├── 2.json          # Task with ID "2"
  ├── .lock           # Lock file for atomic operations
  └── .highwatermark  # Tracks highest task ID issued
```

### Key Path Functions

| Function | Purpose |
|----------|---------|
| `$8()` | Get Claude config dir (`~/.claude`) |
| `MW()` | Get teams dir (`~/.claude/teams`) |
| `i06(teamName)` | Get team directory path |
| `hp4(teamName)` | Alias for `i06` |
| `ik(teamName)` | Get task directory path |
| `gkA(name)` | Normalize team/agent name |
| `Ws(agentName, teamName)` | Get inbox file path |
| `Hy1(teamName, taskId)` | Get task file path |

---

## 3. Team Creation & Deletion

### TeamCreate Tool


#### Input Schema
```typescript
{
  team_name: string;      // Required: name of the team
  description?: string;   // Optional: team description
  agent_type?: string;    // Optional: type for lead agent (default: "team-lead")
}
```

#### Creation Process

1. **Validation**
   - Checks that team_name is provided and non-empty
   - Validates that current session is not already leading a team

2. **Team Name Normalization**
   - Function: `hLY(team_name)` → `gkA(name)`
   - Source: `src/vendor/dom.ts:59987-59989`
   ```typescript
   function gkA(A) {
     return A.replace(/[^a-zA-Z0-9]/g, "-").toLowerCase();
   }
   ```
   - Converts special characters to hyphens and lowercases
   - Example: "My Team!" → "my-team-"

3. **Generate Agent ID**
   - Function: `Pv(agentName, teamName)` 
   - Creates unique identifier for the lead agent

4. **Create Team Config Object**
   ```typescript
   {
     name: normalizedTeamName,
     description: description,
     createdAt: Date.now(),
     leadAgentId: generatedAgentId,
     leadSessionId: currentSessionId,
     members: [
       {
         agentId: leadAgentId,
         name: "team-lead",  // Default agent type
         agentType: agentType || "team-lead",
         model: currentModel,
         joinedAt: Date.now(),
         tmuxPaneId: "",
         cwd: currentWorkingDirectory,
         subscriptions: [],
       }
     ]
   }
   ```

5. **Write Config to Disk**
   - Function: `SLY(teamName, configObject)`
   - Source: `src/vendor/dom.ts:59993-60000`
   ```typescript
   function SLY(A, q) {
     let K = hp4(A);                    // Get team directory path
     CLY(K, { recursive: !0 });         // Create directory (mkdirSync)
     let Y = pkA(K, "config.json");     // Join path
     l8(Y, Q1(q, null, 2));             // Write JSON file (writeFileSync with JSON.stringify)
   }
   ```

6. **Create Task Directory**
   - Function: `N46(taskDir)`
   - Creates corresponding task list directory

7. **Update Session Context**
   - Adds team context to current session state
   - Includes team name, file path, lead agent ID, and teammate info

#### Return Value
```typescript
{
  data: {
    team_name: normalizedName,
    team_file_path: configFilePath,
    lead_agent_id: generatedAgentId
  }
}
```

### TeamDelete Tool


#### Process

1. **Validation**
   - Checks that team has no active members
   - Team lead must terminate all teammates first

2. **Cleanup Operations** (source: `src/vendor/dom.ts:324-350`)
   ```typescript
   // Read team config to find worktree paths
   const config = yX(teamName);
   const worktreePaths = config.members
     .map(m => m.worktreePath)
     .filter(Boolean);
   
   // Delete worktrees
   for (let path of worktreePaths) {
     await wXY(path);  // Remove git worktree
   }
   
   // Delete team directory
   const teamDir = i06(teamName);  // ~/.claude/teams/{team-name}
   await ME4(teamDir, { recursive: true, force: true });
   
   // Delete tasks directory
   const tasksDir = ik(teamName);  // ~/.claude/tasks/{team-name}
   await ME4(tasksDir, { recursive: true, force: true });
   ```

3. **Clear Session Context**
   - Removes team context from current session state

#### Important Constraints

- **MUST terminate all teammates before deletion**
- Automatic cleanup of associated resources:
  - Git worktrees
  - Team directory
  - Task directory
  - Session context

---

## 4. Team Config File Schema

### TeamConfig Interface

**Location**: `~/.claude/teams/{team-name}/config.json`

```typescript
interface TeamConfig {
  name: string;              // Normalized team name
  description?: string;      // Optional description
  createdAt: number;         // Timestamp (Date.now())
  leadAgentId: string;       // UUID of team leader
  leadSessionId: string;     // Session ID of team leader
  members: TeamMember[];     // Array of all team members
  hiddenPaneIds?: string[];  // Optional: hidden tmux panes
}

interface TeamMember {
  agentId: string;           // Unique agent identifier
  name: string;              // Human-readable name (e.g., "team-lead", "researcher")
  agentType: string;         // Type/role (e.g., "team-lead", "engineer", "tester")
  model?: string;            // AI model being used
  joinedAt: number;          // Timestamp when member joined
  tmuxPaneId?: string;       // Tmux pane identifier (if using tmux)
  cwd?: string;              // Working directory
  subscriptions?: any[];     // Subscription info
  prompt?: string;           // Initial prompt for the agent
  color?: string;            // Display color
  planModeRequired?: boolean;// Whether agent requires plan approval
  worktreePath?: string;     // Git worktree path (if using worktrees)
  allowedTools?: string[];   // Whitelist of tools
  disallowedTools?: string[];// Blacklist of tools
}
```

### SessionTeamContext Interface

```typescript
interface SessionTeamContext {
  teamName: string;
  teamFilePath: string;
  leadAgentId: string;
  teammates: {
    [agentId: string]: {
      name: string;
      agentType: string;
      color: string;
      tmuxSessionName: string;
      tmuxPaneId: string;
      cwd: string;
      spawnedAt: number;
    }
  };
}
```

### Example JSON

```json
{
  "name": "my-team",
  "description": "Working on feature X",
  "createdAt": 1705334400000,
  "leadAgentId": "uuid-1234-5678",
  "leadSessionId": "session-abcd",
  "members": [
    {
      "agentId": "uuid-1234-5678",
      "name": "team-lead",
      "agentType": "team-lead",
      "model": "claude-sonnet-4-5",
      "joinedAt": 1705334400000,
      "tmuxPaneId": "",
      "cwd": "/home/user/project",
      "subscriptions": [],
      "color": "blue"
    },
    {
      "agentId": "uuid-researcher",
      "name": "researcher",
      "agentType": "researcher",
      "model": "claude-sonnet-4-5",
      "joinedAt": 1705334500000,
      "cwd": "/home/user/project",
      "color": "cyan",
      "planModeRequired": false,
      "allowedTools": ["Read", "Grep", "Glob"]
    }
  ]
}
```

---

## 5. Agent Types & Spawning

### Agent Types

The system supports four primary agent types:

#### 1. `in_process_teammate`
- Spawned within the same process
- Used for team-based collaboration
- Can be set to idle/active states
- Tracked via `tasks` object in app state

#### 2. `local_agent`
- Runs locally but as a separate task
- Can be killed or brought to foreground
- Tracked in background tasks dialog

#### 3. `remote_agent`
- Remote session execution
- Handles distributed agent execution

#### 4. `local_bash`
- Shell-based execution tasks

### Agent Configuration Properties


```javascript
{
  agentId: string,          // Unique identifier (UUID)
  agentType: string,        // Type identifier (e.g., "engineer", "researcher")
  agentName: string,        // Human-readable name
  teamName: string,         // Team identifier for collaboration
  messages: Array,          // Conversation history
  options: Object,          // Agent-specific configuration
  queryTracking: {          // Request tracking
    chainId: string,        // Request chain identifier
    depth: number           // Recursion depth
  },
  fileReadingLimits: Object,
  userModified: boolean,
  criticalSystemReminder_EXPERIMENTAL: string,
  requireCanUseTool: boolean
}
```

### Spawning Mechanism

#### Entry Point: `qLA` Function


**Function Signature:**
```javascript
qLA(
  mode,                    // Permission mode
  abortSignal,            // Cancellation signal
  undefined,              // Unknown parameter
  stopHooks,              // Stop hook enabled flag
  agentId,                // Agent identifier
  toolUseContext,         // Execution context
  messages,               // Message history
  agentType               // Agent type string
)
```

### Agent ID Generation

Multiple patterns discovered:

1. **`sL()` function** - Custom ID generator
   - Location: `src/tools/tool-4.ts:1114`
   - Usage: `agentId: q?.agentId ?? sL()`

2. **`randomUUID()` from crypto**
   - Location: `src/tools/tool-4.ts:214`
   - Import: `import { randomUUID as Mt4 } from "crypto"`
   - Used for query chain IDs

3. **Query Chain ID Generation**
   ```javascript
   let W = w.queryTracking
     ? { chainId: w.queryTracking.chainId, depth: w.queryTracking.depth + 1 }
     : { chainId: Mt4(), depth: 0 }
   ```
   - Location: `src/tools/tool-4.ts:248-250`

### Member Registration

When a new teammate is spawned, they are added to the config:

```typescript
U.members.push({
  agentId: generatedId,
  name: memberName,
  agentType: agentType,
  model: modelName,
  prompt: promptText,
  color: assignedColor,
  planModeRequired: planModeFlag,
  joinedAt: Date.now(),
  worktreePath: worktreePath,
  subscriptions: []
});

// Write updated config back to disk
cTA(teamName, updatedConfig);
```


---

## 6. Agent Lifecycle

### Lifecycle States


```javascript
// Task states observed:
status: "running" | "pending" | "idle" | "completed"

// Additional state properties:
isIdle: boolean              // Idle flag for teammates
isActive: boolean            // Activity flag
onIdleCallbacks: Function[]  // Callbacks when becoming idle
```

### State Transitions

```
[Creation] → status: "running", isIdle: false
    ↓
[Working] → status: "running", isIdle: false
    ↓
[Idle] → status: "running", isIdle: true
    ↓
[Shutdown] → status: "completed"/"terminated"
```

### Idle Detection

**Function**: `Y8A` (waitForTeammatesToBecomeIdle)


```javascript
function Y8A(A, q) {  // waitForTeammatesToBecomeIdle
  let K = [];
  // Find all non-idle running teammates
  for (let [Y, z] of Object.entries(q.tasks))
    if (z.type === "in_process_teammate" && z.status === "running" && !z.isIdle)
      K.push(Y);
  
  if (K.length === 0) return Promise.resolve();
  
  // Register callbacks for when teammates become idle
  return new Promise((Y) => {
    let z = K.length;
    let w = () => {
      if ((z--, z === 0)) Y();
    };
    // ... callback registration logic
  });
}
```

### Helper Functions

| Function | Purpose |
|----------|---------|
| `hasWorkingInProcessTeammates()` | Returns true if any teammate is running AND not idle |
| `hasActiveInProcessTeammates()` | Returns true if any teammate is running |
| `Kz()` | Check if running as teammate |
| `iM()` | Check if running as team lead |
| `zy1()` | Check if plan mode required |

### Stop Hooks


Agents can execute "Stop" hooks before termination:

#### Hook Events

```javascript
if (
  "hookEvent" in x &&
  (x.hookEvent === "Stop" || x.hookEvent === "SubagentStop")
) {
  // Handle stop hook
}
```

#### Hook Types

1. **`hook_non_blocking_error`** - Error that doesn't stop execution
2. **`hook_error_during_execution`** - Error during hook execution
3. **`hook_success`** - Successful hook execution
4. **`hook_stopped_continuation`** - Hook prevented continuation

#### SubagentStart Hook
- Summary: "When a subagent (Task tool call) is started"
- Input: JSON with `agent_id` and `agent_type`
- Exit code 0: stdout shown to subagent
- Other exit codes: show stderr to user only

#### SubagentStop Hook
- Summary: "Right before a subagent (Task tool call) concludes"
- Input: JSON with `agent_id`, `agent_type`, and `agent_transcript_path`
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to subagent and continue running
- Other exit codes: show stderr to user only

#### TeammateIdle Hook
- Summary: "When a teammate is about to go idle"
- Input: JSON with `teammate_name` and `team_name`
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to teammate and prevent idle
- Other exit codes: show stderr to user only

#### TaskCompleted Hook
- Summary: "When a task is being marked as completed"
- Input: JSON with `task_id`, `task_subject`, `task_description`, `teammate_name`, `team_name`
- Exit code 0: stdout/stderr not shown
- Exit code 2: show stderr to model and prevent task completion
- Other exit codes: show stderr to user only

---

## 7. Permission Modes

The Claude Code CLI implements six distinct permission modes that control how agents interact with the system:

### Mode Definitions

#### 1. `default`
**Standard interactive behavior**
- Prompts user for permission on sensitive operations
- File edits require confirmation
- Default mode for most interactive sessions
- Used when no specific mode is set

#### 2. `acceptEdits`
**Auto-accept file changes**
- Skips permission prompts for file edits
- Still prompts for other sensitive operations
- Useful for trusted automation scenarios

#### 3. `bypassPermissions`
**Full bypass mode**
- Auto-accepts ALL operations without prompting
- Most permissive mode
- Can be disabled via:
  - Environment: `CLAUDE_CODE_BYPASS_PERMISSIONS_MODE`
  - Remote settings: `permissions.disableBypassPermissionsMode`
- Requires `isBypassPermissionsModeAvailable` flag

#### 4. `plan`
**Analysis-only mode**
- Agent can read/analyze but NOT execute changes
- Prevents file edits, commits, or system modifications
- Exception: Can edit the plan file itself
- Automatically set when `plan_mode_required` is true

#### 5. `delegate`
**Delegate decisions to another agent**
- Forwards permission decisions to a parent/lead agent
- Used in team hierarchies

#### 6. `dontAsk`
**Similar to bypassPermissions**
- Skip permission prompts

### Core Interface

```typescript
interface ToolPermissionContext {
  mode: "default" | "acceptEdits" | "bypassPermissions" | "plan" | "delegate" | "dontAsk";
  isBypassPermissionsModeAvailable: boolean;
  additionalWorkingDirectories: Map<string, any>;
  alwaysAllowRules: Record<string, any>;
  alwaysDenyRules: Record<string, any>;
  alwaysAskRules: Record<string, any>;
  shouldAvoidPermissionPrompts?: boolean;
}
```

### Mode Resolution for Teammates

```typescript
// From src/conversation/session-1.ts:8061
let mode = isTeammate() && isPlanModeRequired() ? "plan" : "default";

// From src/agents/startdeferredprefetches-1.ts:1434
mode: p8() && Akq().isPlanModeRequired() ? "plan" : e1.mode
```

### Permission Check Flow

1. Check if mode is `bypassPermissions` or `acceptEdits`
2. If true, auto-allow with decision reason: `{ type: "mode", mode: context.mode }`
3. Otherwise, check rule-based permissions
4. Finally, prompt user if needed

```typescript
if (z.toolPermissionContext.mode === "bypassPermissions" ||
    (z.toolPermissionContext.mode === "acceptEdits" &&
     z.toolPermissionContext.isBypassPermissionsModeAvailable))
  return {
    behavior: "allow",
    decisionReason: { type: "mode", mode: z.toolPermissionContext.mode }
  };
```

### CLI Configuration

#### Command-Line Flags
```bash
# Permission mode
claude --permission-mode plan
claude --permission-mode bypassPermissions
claude --permission-mode acceptEdits

# Task with specific mode
claude -p "fix lint errors" --permission-mode acceptEdits
```

#### Environment Variables
```bash
# Enable bypass permissions
export CLAUDE_CODE_BYPASS_PERMISSIONS_MODE=true

# Force plan mode for all agents
export CLAUDE_CODE_PLAN_MODE_REQUIRED=true
```

---

## 8. Plan Mode

### Overview

Plan mode is a special workflow where an agent creates an implementation plan before execution. The team lead must approve the plan before the agent can proceed.

### Configuration

```typescript
interface TeammateConfig {
  agentName: string;
  teamName: string;
  prompt: string;
  color: string;
  planModeRequired: boolean;  // If true, agent starts in plan mode
  model?: string;
}
```

### Plan Mode Detection

```typescript
function isPlanModeRequired() {
  let context = getTeammateContext();
  if (context) return context.planModeRequired;
  if (BT !== null) return BT.planModeRequired;
  return process.env.CLAUDE_CODE_PLAN_MODE_REQUIRED === "true";
}
```

### Plan Approval Flow (5 Steps)

#### 1. Agent Creates Plan
- Agent works in `plan` mode (read-only except plan file)
- Creates plan at designated file path
- Documents implementation steps

#### 2. Agent Calls ExitPlanMode
- Tool: `ExitPlanMode`
- Validates plan exists and is complete
- Sends plan approval request to team lead

```typescript
// ExitPlanMode description
"IMPORTANT: Only use this tool when the task requires planning 
the implementation steps of a task that requires writing code.

Examples:
1. Initial task: 'Search for and understand vim mode' 
   → Do NOT use ExitPlanMode (research task)
2. Initial task: 'Implement yank mode for vim' 
   → Use ExitPlanMode after planning implementation steps
"
```

#### 3. Plan Approval Request Schema

```typescript
type PlanApprovalRequest = {
  type: "plan_approval_request";
  from: string;          // Agent name
  timestamp: string;     // ISO timestamp
  planFilePath: string;  // Path to plan file
  planContent: string;   // Full plan text
  requestId: string;     // Unique request ID
};
```

#### 4. Plan Approval Response Schema

```typescript
type PlanApprovalResponse = {
  type: "plan_approval_response";
  requestId: string;        // Matches request
  approved: boolean;        // true = approved, false = rejected
  feedback?: string;        // Optional feedback on rejection
  timestamp: string;        // ISO timestamp
  permissionMode?: string;  // Optional mode to set after approval
};
```

#### 5. Post-Approval
- If approved: Agent exits plan mode and can execute
- If rejected: Agent receives feedback and revises plan
- Agent must call ExitPlanMode again after revisions

### Constraints in Plan Mode

- **Cannot** make file edits (except plan file)
- **Cannot** run write operations
- **Cannot** make commits or push changes
- **Can** read files, search codebase, analyze code
- **Can** edit the designated plan file

### Plan Approval Response Implementation

**Approve**: `lLY()` function
**Reject**: `iLY()` function

**Restrictions**:
- Only team lead can approve/reject (checked via `iM()`)
- Teammates cannot approve their own or other plans

**Approval Process**:
1. Validates sender is team lead
2. Determines permission mode to grant (avoids plan/delegate modes)
3. Creates approval response with permission mode
4. Writes to requesting agent's mailbox
5. Agent receives approval and exits plan mode

**Rejection Process**:
1. Validates sender is team lead
2. Creates rejection with feedback
3. Writes to requesting agent's mailbox
4. Agent can revise plan based on feedback

---

## 9. Tool Restrictions

### Per-Agent Tool Filtering

```typescript
interface AgentDefinition {
  name: string;
  description: string;
  allowedTools: string[];  // Whitelist of tools
  disallowedTools?: string[];  // Blacklist of tools
  model?: string;
  systemPrompt: string;
}
```

### Command-Level Tool Restrictions

```typescript
{
  allowedTools: z.array(z.string())
    .optional()
    .describe("Array of allowed tool names. If omitted, inherits all tools from parent"),
  disallowedTools: z.array(z.string())
    .optional()
    .describe("Array of tool names to explicitly disallow for this agent")
}
```

### Tool Name Format Patterns

Tools can be restricted with patterns:
- Simple name: `"Edit"`, `"Read"`
- With path pattern: `"Read(~/**)"`
- Git commands: `"Bash(git:*)"` 
- Multiple: `"Bash,Edit,Read"`

### Tool Resolution Order

1. Start with base tool set
2. Apply `allowedTools` filter (if specified)
3. Remove `disallowedTools` (if specified)
4. Inherit from parent if not specified

### Agent Type Definitions

```typescript
type AgentType = 
  | "engineer"           // Full-stack implementation
  | "reviewer"           // Code review
  | "tester"            // Test writing/execution
  | "architect"         // System design
  | "deployer"          // Deployment operations
  | "researcher"        // Investigation/exploration
  | "teammate"          // Generic team member
  | "general-purpose";  // Default type
```

### Task Spawning with Tool Restrictions

```typescript
interface TaskToolInput {
  prompt: string;              // Task description
  agent?: string;              // Agent type to use
  allowedTools?: string[];     // Tool whitelist for this task
  mode?: PermissionMode;       // Permission mode override
  plan_mode_required?: boolean;// Force plan mode
}
```

---

## 10. Task Management System

### Storage Architecture

#### File System Structure

```
${$8()}/tasks/${teamName}/
  ├── 1.json          # Task with ID "1"
  ├── 2.json          # Task with ID "2"
  ├── 3.json          # Task with ID "3"
  ├── .lock           # Lock file for atomic operations
  └── .highwatermark  # Tracks highest task ID issued
```

**Key Functions** (`src/core/filesystem-1.ts`):
- `ik(teamName)` - Returns task directory path: `${$8()}/tasks/${sanitized(teamName)}/`
- `Hy1(teamName, taskId)` - Returns task file path: `${taskDir}/${sanitized(taskId)}.json`
- `nO1(str)` - Sanitizes names by replacing non-alphanumeric chars with hyphens
- `nM()` - Gets current task list ID from env var, team context, or session ID

### Task JSON Schema

```typescript
interface Task {
  id: string;              // Unique task ID (numeric string: "1", "2", etc.)
  subject: string;         // Short task title/summary
  description: string;     // Detailed task description
  status: TaskStatus;      // Current task status
  owner?: string;          // Agent ID that owns/claimed this task
  blocks: string[];        // Array of task IDs this task blocks
  blockedBy: string[];     // Array of task IDs blocking this task
  metadata?: {
    _internal?: boolean;   // Internal system task flag
    [key: string]: any;    // Additional metadata
  };
}

type TaskStatus = "pending" | "in_progress" | "completed" | "deleted";
```

**Schema Validation:**
- Schema validator referenced as `jG5` (likely Zod schema)
- Validates task structure when reading from disk
- Located in `src/core/filesystem-1.ts` line ~1546

### Core Operations

#### Task Create (`rO1`)

**Function:** `rO1(teamName, taskData)`

**Implementation:**
```typescript
function rO1(teamName, taskData) {
  let lockFile = gt8(teamName);  // Get lock file path
  let release;
  try {
    release = wy1.default.lockSync(lockFile);  // Acquire lock
    let highestId = GG5(teamName);  // Get max(diskId, highwatermark)
    let newId = String(highestId + 1);
    let task = { id: newId, ...taskData };
    let taskPath = Hy1(teamName, newId);
    l8(taskPath, JSON.stringify(task, null, 2));  // Write task file
    iO1();  // Notify listeners
    return newId;
  } finally {
    if (release) release();  // Release lock
  }
}
```

**Features:**
- Thread-safe task creation using file locks
- Auto-incrementing numeric task IDs
- Persists to individual JSON files
- Triggers change notifications

#### Task Read (`OU`)

**Function:** `OU(teamName, taskId)`

**Implementation:**
```typescript
function OU(teamName, taskId) {
  let taskPath = Hy1(teamName, taskId);
  try {
    let content = xt8(taskPath, "utf-8");  // Read file
    let json = JSON.parse(content);
    let validated = jG5.safeParse(json);  // Validate schema
    if (!validated.success) {
      log(`Task ${taskId} failed validation: ${validated.error.message}`);
      return null;
    }
    return validated.data;
  } catch (error) {
    if (error.code === "ENOENT") return null;  // Task doesn't exist
    log(`Failed to read task ${taskId}: ${error.message}`);
    return null;
  }
}
```

**Features:**
- Reads individual task from JSON file
- Schema validation on read
- Returns `null` for missing/invalid tasks
- Error logging

#### Task Update (`QC`)

**Function:** `QC(teamName, taskId, updates)`

**Implementation:**
```typescript
function QC(teamName, taskId, updates) {
  let existingTask = OU(teamName, taskId);
  if (!existingTask) return null;
  
  let updatedTask = { ...existingTask, ...updates, id: taskId };
  let taskPath = Hy1(teamName, taskId);
  l8(taskPath, JSON.stringify(updatedTask, null, 2));
  iO1();  // Notify listeners
  return updatedTask;
}
```

**Features:**
- Merge updates with existing task data
- Preserves task ID (cannot be changed)
- Notifies listeners of changes
- Returns updated task object

#### Task List (`zX`)

**Function:** `zX(teamName)`

**Implementation:**
```typescript
function zX(teamName) {
  let taskDir = ik(teamName);
  if (!hn(taskDir)) return [];  // Directory doesn't exist
  
  let files = H8A(taskDir);  // Read directory
  let tasks = [];
  for (let file of files) {
    if (!file.endsWith(".json")) continue;
    let taskId = file.replace(".json", "");
    let task = OU(teamName, taskId);
    if (task) tasks.push(task);
  }
  return tasks;
}
```

**Features:**
- Reads all `.json` files from task directory
- Skips non-JSON files (.lock, .highwatermark)
- Returns array of validated task objects
- Filters out invalid/corrupted tasks

#### Task Delete (`T46`)

**Function:** `T46(teamName, taskId)`

**Features:**
- Deletes task JSON file from disk
- Updates highwatermark to prevent ID reuse
- Cleans up dependency references in other tasks
- Notifies listeners

### Task States & Transitions

#### State Definitions

1. **`pending`**
   - Task has been created but not yet claimed
   - No owner assigned
   - Available for agents to claim

2. **`in_progress`**
   - Task has been claimed by an agent
   - Has `owner` field set to agent ID
   - Agent is actively working on it

3. **`completed`**
   - Task has been finished
   - No longer blocks other tasks
   - Cannot be claimed by agents

4. **`deleted`**
   - Task has been removed (not commonly used in code)
   - Typically tasks are deleted from disk rather than marked deleted

#### State Diagram

```
[Create] → pending
            ↓ [Claim]
        in_progress
            ↓ [Complete]
         completed
            ↓ [Delete]
         [Removed from disk]
```

### Dependencies (blocks/blockedBy)

#### Dependency Fields

- **`blocks: string[]`** - Tasks that cannot start until this task completes
- **`blockedBy: string[]`** - Tasks that must complete before this task can start

#### Adding Dependency (`_8A`)

```typescript
function _8A(teamName, blockingTaskId, blockedTaskId) {
  let blockingTask = OU(teamName, blockingTaskId);
  let blockedTask = OU(teamName, blockedTaskId);
  if (!blockingTask || !blockedTask) return false;
  
  // Update blocking task's "blocks" array
  if (!blockingTask.blocks.includes(blockedTaskId)) {
    QC(teamName, blockingTaskId, { 
      blocks: [...blockingTask.blocks, blockedTaskId] 
    });
  }
  
  // Update blocked task's "blockedBy" array
  if (!blockedTask.blockedBy.includes(blockingTaskId)) {
    QC(teamName, blockedTaskId, { 
      blockedBy: [...blockedTask.blockedBy, blockingTaskId] 
    });
  }
  
  return true;
}
```

**Dependency Validation:**
- When claiming a task, system checks if all `blockedBy` tasks are completed
- Only tasks with no incomplete blockers can be claimed
- Circular dependencies are not automatically prevented (application-level concern)

### Ownership and Claiming

#### Claiming Mechanism (`J8A`)

**Function:** `J8A(teamName, taskId, agentId, options)`

**Claim Failure Reasons:**
1. `task_not_found` - Task doesn't exist
2. `already_claimed` - Another agent owns the task
3. `already_resolved` - Task is already completed
4. `blocked` - Task has incomplete dependencies
5. `agent_busy` - Agent already has active tasks (when `checkAgentBusy: true`)

**Process:**
1. Lock task file
2. Read task
3. Check if already claimed
4. Check if completed
5. Check dependencies
6. Claim successful → update owner field

#### Agent Busy Check (`ZG5`)

**Function:** `ZG5(teamName, taskId, agentId)`

**Features:**
- Uses team-level lock instead of task-level lock
- Checks if agent already owns other incomplete tasks
- Prevents agents from claiming multiple tasks simultaneously
- Returns `busyWithTasks` array if agent is busy

#### Automatic Unassignment on Shutdown (`In`)

**Function:** `In(teamName, agentId, agentName, reason)`

**Use Case:** When agent shuts down or terminates

**Process:**
```typescript
function In(teamName, currentAgentId, agentName, reason) {
  // Find all tasks owned by the agent
  let ownedTasks = zX(teamName).filter(
    task => task.status !== "completed" && 
            (task.owner === currentAgentId || task.owner === agentName)
  );
  
  // Unassign all tasks
  for (let task of ownedTasks) {
    QC(teamName, task.id, { owner: undefined, status: "pending" });
  }
  
  if (ownedTasks.length > 0) {
    log(`Unassigned ${ownedTasks.length} task(s) from ${agentName}`);
  }
  
  // Build notification message
  let message = `${agentName} ${reason === "terminated" ? "was terminated" : "has shut down"}.`;
  if (ownedTasks.length > 0) {
    let taskList = ownedTasks.map(t => `#${t.id} "${t.subject}"`).join(", ");
    message += ` ${ownedTasks.length} task(s) were unassigned: ${taskList}. Use TaskList to check availability and TaskUpdate with owner to reassign them to idle teammates.`;
  }
  
  return {
    unassignedTasks: ownedTasks.map(t => ({ id: t.id, subject: t.subject })),
    notificationMessage: message
  };
}
```

**Features:**
- Automatically unassigns tasks when agent goes offline
- Resets task status to `pending`
- Clears `owner` field
- Returns notification message for team lead

### Concurrency Control (File Locking)

**File-based Locking:**
```typescript
import wy1 from "proper-lockfile";  // Assumed based on usage pattern

// Task-level lock (for claim/update)
let release = wy1.default.lockSync(taskPath);
try {
  // Perform operation
} finally {
  if (release) release();
}

// Team-level lock (for create/busy-check)
let lockFile = gt8(teamName);  // Returns ${taskDir}/.lock
let release = wy1.default.lockSync(lockFile);
try {
  // Perform operation
} finally {
  if (release) release();
}
```

**Lock Granularity:**
- **Task-level locks** - Used for claiming and updating individual tasks
- **Team-level locks** - Used for creating tasks and checking agent availability
- **.lock file** - Persistent lock file in task directory

### High Watermark System

#### Purpose
Prevents task ID reuse even after deletion.

#### Implementation

**File:** `.highwatermark` in task directory

**Read Watermark (`O8A`):**
```typescript
function O8A(teamName) {
  let watermarkPath = Ft8(teamName);  // ${taskDir}/.highwatermark
  try {
    let content = xt8(watermarkPath, "utf-8").trim();
    let value = parseInt(content, 10);
    return isNaN(value) ? 0 : value;
  } catch {
    return 0;
  }
}
```

**Get Max ID (`GG5`):**
```typescript
function GG5(teamName) {
  let maxOnDisk = Ut8(teamName);  // Scan .json files
  let watermark = O8A(teamName);   // Read .highwatermark
  return Math.max(maxOnDisk, watermark);
}
```

**Why It Exists:**
- Prevents ID collision after task deletion
- Example: Task #5 deleted → watermark prevents reusing #5
- Ensures task IDs are monotonically increasing
- Critical for referential integrity in logs/history

---

## 11. Inter-Agent Messaging

### SendMessage Tool Overview

**Tool Name**: `SendMessage` (internal name: `OB`)  
**Module**: `qd4` (SendMessageTool)

The SendMessage tool is the primary communication mechanism for agents in teams. It supports five distinct message types for different communication patterns.

**Tool Configuration:**
- **Concurrency Safe**: `false` - Messages cannot be sent concurrently
- **Read Only**: `true` for `message` and `broadcast`, `false` for protocol messages
- **Permissions**: Always allowed (`behavior: "allow"`)
- **Max Result Size**: 100,000 characters
- **Enabled**: Only when teams are enabled (`p8()` check)

### Message Types (5 Types)

```typescript
type SendMessageInput = 
  | { type: "message", recipient: string, content: string, summary: string }
  | { type: "broadcast", content: string, summary: string }
  | { type: "shutdown_request", recipient: string, content?: string }
  | { type: "shutdown_response", request_id: string, approve: boolean, content?: string }
  | { type: "plan_approval_response", request_id: string, approve: boolean, recipient: string, content?: string }
```

#### 1. Direct Message ("message")

**Implementation**: `ULY()` function

**Process**:
1. Gets current app state and team context
2. Resolves sender name (from `B5()` or defaults to "teammate" or team-lead)
3. Normalizes recipient name via `ep4()`
4. Writes message to recipient's inbox using `X9()` (writeToMailbox)
5. Returns success with routing metadata

**Response**:
```typescript
{
  data: {
    success: true,
    message: "Message sent to {recipient}'s inbox",
    routing: {
      sender: string,
      senderColor: string,
      target: string,
      targetColor: string,
      summary: string,
      content: string
    }
  }
}
```

#### 2. Broadcast ("broadcast")

**Implementation**: `gLY()` function

**Process**:
1. Validates team context exists
2. Retrieves team roster via `E31(teamName)`
3. Filters out sender from recipient list
4. Calls `X9()` (writeToMailbox) for each teammate
5. Returns list of recipients

**Key Behavior**:
- Sends separate message to EVERY teammate (N deliveries for N teammates)
- Expensive operation - scales linearly with team size
- Skips sender automatically
- Returns empty recipients list if sender is only team member

#### 3. Shutdown Request ("shutdown_request")

**Implementation**: `pLY()` function

**Process**:
1. Generates unique request ID via `Vj1("shutdown", recipient)`
2. Creates shutdown request message with `pj1()`
3. Writes to recipient's mailbox
4. Returns request ID for tracking

**Message Format**:
```typescript
{
  type: "shutdown_request",
  requestId: string,
  from: string,
  reason: string,
  timestamp: string
}
```

#### 4. Shutdown Response ("shutdown_response")

**Approve**: `dLY()` function
**Reject**: `cLY()` function

**Approval Process**:
1. Retrieves team context and agent info
2. Gets tmux pane ID and backend type from team roster
3. Creates shutdown approved message with `cNA()`
4. Writes to team-lead's mailbox
5. If in-process teammate: aborts the AbortController
6. If tmux teammate: calls `RK()` to exit process

**Rejection Process**:
1. Creates shutdown rejected message with `lNA()`
2. Writes to team-lead's mailbox
3. Returns rejection confirmation

#### 5. Plan Approval Response ("plan_approval_response")

**Approve**: `lLY()` function
**Reject**: `iLY()` function

**Restrictions**:
- Only team lead can approve/reject (checked via `iM()`)
- Teammates cannot approve their own or other plans

### Message Delivery Mechanism


#### `writeToMailbox()` - Function `X9`

The primary message delivery function.

**Process**:
1. Ensures inbox directory exists via `RWY()`
2. Gets inbox file path via `Ws(agentName, teamName)`
3. Creates inbox file if doesn't exist
4. Acquires file lock using `cm1.lockSync()` (proper-lockfile)
5. Reads current inbox contents
6. Appends new message with `read: false`
7. Writes updated inbox back to file
8. Releases lock in finally block

**File Structure**: 
- Path: `{projectRoot}/.claude/teams/{teamName}/inboxes/{agentName}.json`
- Format: JSON array of message envelopes

**Thread Safety**:
- Uses file locking via `proper-lockfile` package
- Lock file: `{inboxPath}.lock`
- Ensures atomic read-modify-write operations

#### `readMailbox()` - Function `ip`

Reads all messages from an agent's inbox.

**Process**:
1. Gets inbox path via `Ws()`
2. Returns empty array if file doesn't exist
3. Reads file contents and parses JSON
4. Returns array of messages
5. Handles errors gracefully

#### `readUnreadMessages()` - Function `M31`

Filters for unread messages only.

**Process**:
1. Calls `readMailbox()` to get all messages
2. Filters messages where `read === false`
3. Returns unread messages

#### `markMessagesAsRead()` - Function `im1`

Marks all messages in inbox as read.

#### `markMessageAsReadByIndex()` - Function `lm1`

Marks specific message as read by index.

### Inbox Polling System


The inbox poller runs on a timer and automatically delivers messages from the filesystem to the agent's session state.

**Key Components:**

#### `InboxPoller` Hook

**Trigger**:
- React hook that polls inbox periodically
- Only runs when agent is active (`A` parameter)

**Process**:
1. Gets current state via `w.getState()`
2. Retrieves agent name via `LT6(state)`
3. Calls `M31()` to read unread messages
4. Categorizes messages by type:
   - Permission requests
   - Permission responses
   - Sandbox permission requests
   - Sandbox permission responses
   - Shutdown requests
   - Shutdown approvals
   - Team permission updates
   - Mode set requests
   - Regular messages (DMs/broadcasts)

5. Processes each category:
   - **Plan approvals**: Exits plan mode if approved by team lead
   - **Permission requests**: Adds to permission queue if team lead
   - **Shutdown requests**: Creates shutdown prompt
   - **Regular messages**: Formats and injects into conversation

6. Marks all messages as read via `im1()`

### Message Injection into Conversation

**Function**: `formatTeammateMessages()` - `CWY`

Formats messages for injection into agent's system prompt.

**Format**:
```xml
<teammate_message teammate_id="{from}" color="{color}" summary="{summary}">
{message_text}
</teammate_message>
```

**Multiple Messages**:
```xml
<teammate_message teammate_id="researcher" color="cyan" summary="Found API pattern">
Here's the authentication pattern I found...
</teammate_message>

<teammate_message teammate_id="tester" color="green" summary="Tests passing">
All unit tests are now passing after the fix.
</teammate_message>
```

Messages are joined with double newlines for readability.

### Idle Notifications

#### Creation

**Function**: `createIdleNotification()` - `nm1`

**Structure**:
```typescript
{
  type: "idle_notification",
  from: string,                    // Agent name
  timestamp: string,               // ISO timestamp
  idleReason?: string,             // Why agent became idle
  summary?: string,                // Brief summary
  completedTaskId?: string,        // If task completed
  completedStatus?: string,        // "success" | "failed"
  failureReason?: string          // If failed, why
}
```

#### Detection

**Function**: `isIdleNotification()` - `rm1`

Parses message text and checks if `type === "idle_notification"`.

### DM Privacy and Peer Collaboration

#### Direct Message Privacy

**Key Principle**: Direct messages are PRIVATE between sender and recipient.

**Visibility Rules**:
1. Only sender and recipient can see DM content
2. Team lead does NOT see DMs between other teammates
3. Other teammates do NOT see DMs not addressed to them

**Implementation**:
- Each agent has separate inbox file
- Inbox poller only reads current agent's inbox
- No cross-agent message access

#### Peer Collaboration Summary

**Function**: `getLastPeerDmSummary()` - `sm1`

Extracts summary of last DM sent to another peer for UI display.

**Process**:
1. Scans conversation messages in reverse
2. Looks for assistant message with SendMessage tool use
3. Filters for `type === "message"` (not to team-lead)
4. Extracts recipient and summary/content
5. Returns formatted string: `"[to {recipient}] {summary}"`

**Purpose**: 
- Shows in UI what agent is communicating with peers
- Provides visibility to team lead without exposing full content
- Helps team lead understand agent coordination

### Message Envelope Format

#### Filesystem Message Envelope

**Format**: Stored in `{agentName}.json` inbox files

```typescript
interface InboxMessage {
  from: string;              // Sender's agent name
  text: string;              // Message content (may be JSON for protocol messages)
  timestamp: string;         // ISO 8601 timestamp
  read: boolean;             // Read status
  color?: string;            // Sender's UI color
  summary?: string;          // 5-10 word summary for UI
}
```

**Example**:
```json
[
  {
    "from": "researcher",
    "text": "I found the authentication pattern in auth/middleware.ts",
    "summary": "Found auth pattern",
    "timestamp": "2025-01-15T10:30:00.000Z",
    "read": false,
    "color": "cyan"
  }
]
```

#### Session State Message Format

**Format**: After inbox poller processes messages

```typescript
interface SessionInboxMessage {
  id: string;                // Unique message ID (generated)
  from: string;              // Sender's agent name
  text: string;              // Message content
  timestamp: string;         // ISO 8601 timestamp
  status: "pending";         // Always "pending" when added
  color?: string;            // Sender's UI color
  summary?: string;          // 5-10 word summary for UI
}
```

**Location in State**: `state.inbox.messages[]`

### Protocol Message Formats (8+ Types)

All protocol messages are JSON-stringified and stored in the `text` field.

#### Shutdown Request
```typescript
{
  type: "shutdown_request",
  requestId: string,
  from: string,
  reason: string,
  timestamp: string
}
```

#### Shutdown Approved
```typescript
{
  type: "shutdown_approved",
  requestId: string,
  from: string,
  timestamp: string,
  paneId?: string,
  backendType?: "in-process" | "tmux"
}
```

#### Shutdown Rejected
```typescript
{
  type: "shutdown_rejected",
  requestId: string,
  from: string,
  reason: string,
  timestamp: string
}
```

#### Plan Approval Request
```typescript
{
  type: "plan_approval_request",
  from: string,
  timestamp: string,
  planFilePath: string,
  planContent: string,
  requestId: string
}
```

#### Plan Approval Response
```typescript
{
  type: "plan_approval_response",
  requestId: string,
  approved: boolean,
  permissionMode?: string,    // If approved
  feedback?: string,          // If rejected
  timestamp: string
}
```

#### Idle Notification
```typescript
{
  type: "idle_notification",
  from: string,
  timestamp: string,
  idleReason?: string,
  summary?: string,
  completedTaskId?: string,
  completedStatus?: "success" | "failed",
  failureReason?: string
}
```

#### Task Completed
```typescript
{
  type: "task_completed",
  from: string,
  taskId: string,
  taskSubject: string,
  timestamp: string
}
```

#### Permission Request
```typescript
{
  type: "permission_request",
  request_id: string,
  agent_id: string,
  tool_name: string,
  tool_use_id: string,
  description: string,
  input: any,
  timestamp: string
}
```

#### Permission Response
```typescript
{
  type: "permission_response",
  request_id: string,
  decision: "approved" | "rejected",
  resolvedBy: "leader" | "worker",
  updatedInput?: any,
  permissionUpdates?: any,
  feedback?: string,
  timestamp: string
}
```

### Message Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ Agent A (Sender)                                                │
│                                                                 │
│  1. Calls SendMessage tool                                      │
│     { type: "message", recipient: "agent-b", content: "..."}  │
│                                                                 │
│  2. Tool execution: ULY() function                              │
│     - Resolves sender name (B5)                                 │
│     - Normalizes recipient name (ep4)                           │
│     - Gets sender color (R$)                                    │
│                                                                 │
│  3. writeToMailbox (X9)                                         │
│     - Gets inbox path: .claude/teams/{team}/inboxes/agent-b.json│
│     - Acquires file lock                                        │
│     - Reads current inbox                                       │
│     - Appends message envelope { from, text, timestamp, ... }   │
│     - Writes back to file                                       │
│     - Releases lock                                             │
│                                                                 │
│  4. Returns success response                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Filesystem
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Inbox File: agent-b.json                                        │
│                                                                 │
│ [                                                               │
│   {                                                             │
│     "from": "agent-a",                                         │
│     "text": "Message content...",                             │
│     "summary": "Brief summary",                               │
│     "timestamp": "2025-01-15T10:30:00.000Z",                   │
│     "read": false,                                             │
│     "color": "cyan"                                            │
│   }                                                             │
│ ]                                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Polling
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Agent B (Recipient)                                             │
│                                                                 │
│  1. InboxPoller runs (periodic timer)                           │
│     - Calls M31() to read unread messages                       │
│     - Gets messages from agent-b.json                           │
│                                                                 │
│  2. Message categorization                                      │
│     - Checks message type (protocol vs regular)                 │
│     - Routes to appropriate handler                             │
│                                                                 │
│  3. For regular messages:                                       │
│     - Adds to state.inbox.messages[]                            │
│     - Formats via CWY() for system prompt                       │
│     - Injects into next API call as system message              │
│                                                                 │
│  4. Marks messages as read                                      │
│     - Calls im1() to mark all as read                           │
│     - Updates agent-b.json                                      │
│                                                                 │
│  5. Agent sees message in next turn                             │
│     - Formatted as <teammate_message> in system prompt          │
│     - Agent can respond or take action                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. UI & Status Rendering

### Agent Type Discriminators in UI


Two main agent types in the UI:

#### `local_agent`
```typescript
case "local_agent": {
  let w = YY(K.description, z, !0);
  let H = K.status === "completed" ? "done" : void 0;
  let $ = K.status === "completed" && !K.notified ? ", unread" : void 0;
  let O = B0.createElement(BP1, { status: K.status, label: H, suffix: $ });
  // ...
}
```

#### `in_process_teammate`
```typescript
case "in_process_teammate": {
  let w = K.shutdownRequested
    ? "stopping"
    : K.awaitingPlanApproval
      ? "awaiting approval"
      : K.isIdle
        ? "idle"
        : K.progress
          ? K.progress
          : "running";
  // ...
}
```

### Teammate-Specific UI States


Helper functions for filtering agents:
```typescript
function xHz(A) {
  return A.type === "local_agent" && !EZ1(A.status);
}

function bHz(A) {
  return A.type === "in_process_teammate" && A.status === "running";
}

function uHz(A) {
  return A.type === "in_process_teammate";
}

function BHz(A) {
  return A.name !== "team-lead";
}
```

### Status Indicators

- `"running"`: Active execution
- `"idle"`: Waiting for input
- `"stopping"`: Shutdown requested
- `"awaiting approval"`: Plan mode approval pending
- `"completed"`: Task finished

```typescript
f, { color: "warning", bold: !0 }, " ",
"Waiting for team lead approval"
```

### Agent Colors

```typescript
let agentSpawnContext = {
  agentColor: Y.finalAgent.color,
  // ...
};
```


Agent colors are used throughout the UI for:
- Agent name rendering
- Status indicators
- Message attribution
- Task ownership visualization

### Notification System


```typescript
function hq() {
  let A = E6((w) => w.notifications.queue.length);
  let q = b7();
  let K = mG1.useCallback(() => {
    q((w) => {
      let H = fgY(w.notifications.queue);
      if (w.notifications.current !== null || !H) return w;
      return {
        ...w,
        notifications: {
          queue: w.notifications.queue,
          current: H,
        },
      };
    });
  }, [q]);
  // ...
}
```

Notifications are used for:
- Plugin installation status
- Model updates (Sonnet 4.5, Opus 4.5)
- Team member messages
- Idle state transitions
- Task completions

---

## 13. System Prompts for Teams

### Three-Tier Prompt Injection


The system prompt builder supports three modes:

1. **Override** (`overrideSystemPrompt`): Completely replaces the default system prompt
2. **Custom** (`customSystemPrompt`): Uses agent's `getSystemPrompt()` method
3. **Append** (`appendSystemPrompt`): Adds content to the end of the system prompt

**Pattern**: override > custom > default + append

```typescript
function buildSystemPrompts({
  toolUseContext: q,
  customSystemPrompt: K,
  defaultSystemPrompt: Y,
  appendSystemPrompt: z,
  overrideSystemPrompt: w,
}) {
  if (w) return [w];
  let H = A
    ? T0(A)
      ? A.getSystemPrompt({ toolUseContext: { options: q.options } })
      : A.getSystemPrompt()
    : void 0;
  if (A?.memory)
    l("tengu_agent_memory_loaded", {
      ...{},
      scope: A.memory,
      isMainLoopAgent: !0,
    });
  return [...(H ? [H] : K ? [K] : Y), ...(z ? [z] : [])];
}
```

### Team-Specific Prompt Sections


Key sections injected into team member system prompts:

#### TeamCreate Tool
```json
{
  "team_name": "my-project",
  "description": "Working on feature X"
}
```

This creates:
- Team file: `~/.claude/teams/{team-name}.json`
- Task list directory: `~/.claude/tasks/{team-name}/`

#### Team Workflow Instructions
1. **Create a team** with TeamCreate
2. **Create tasks** using Task tools (TaskCreate, TaskList, etc.)
3. **Spawn teammates** using Task tool with `team_name` and `name` parameters
4. **Assign tasks** using TaskUpdate with `owner`
5. **Teammates work on assigned tasks**
6. **Teammates go idle between turns** (automatic)
7. **Shutdown team** via SendMessage with `type: "shutdown_request"`

#### Automatic Message Delivery
- Messages from teammates are automatically delivered
- No manual inbox checking needed
- Messages appear as new conversation turns
- If busy (mid-turn), messages are queued
- UI shows brief notification with sender's name

#### Teammate Idle State Guidance
- Teammates go idle after every turn (normal and expected)
- Idle teammates can receive messages
- Idle notifications are automatic
- Don't treat idle as an error
- Peer DM visibility in idle notifications

#### Team Discovery
- Team config location: `~/.claude/teams/{team-name}/config.json`
- Contains `members` array with:
  - `name`: Human-readable name (use for messaging)
  - `agentId`: Unique identifier (reference only)
  - `agentType`: Role/type of agent

**IMPORTANT**: Always refer to teammates by NAME, not agentId.

#### Task List Coordination
- Shared task list: `~/.claude/tasks/{team-name}/`
- Check TaskList after completing each task
- Claim unassigned tasks with TaskUpdate (set `owner`)
- Prefer tasks in ID order (lowest first)
- Create new tasks when identifying additional work
- Mark completed with TaskUpdate
- Coordinate by reading task list status

### Subagent Hooks


Hook events for subagents:

- **SubagentStart**: When a subagent (Task tool call) is started
- **SubagentStop**: Right before a subagent (Task tool call) concludes
- **TeammateIdle**: When a teammate is about to go idle
- **TaskCompleted**: When a task is being marked as completed

(See section 6 for detailed hook specifications)

---

## 14. Telemetry Events

### Agent Events

#### `tengu_agent_created`


```typescript
l("tengu_agent_created", {
  agent_type: Y.finalAgent.agentType,
  generation_method: Y.wasGenerated ? "generated" : "manual",
  source: Y.location,
  tool_count: Y.finalAgent.tools?.length ?? "all",
  has_custom_model: !!Y.finalAgent.model,
  has_custom_color: !!Y.finalAgent.color,
  has_memory: !!Y.finalAgent.memory,
  memory_scope: Y.finalAgent.memory ?? "none",
  ...(J ? { opened_in_editor: !0 } : {}),
});
```

#### `tengu_agent_definition_generated`


```typescript
l("tengu_agent_definition_generated", { 
  agent_identifier: M.identifier 
});
```

#### `tengu_agent_memory_loaded`


```typescript
l("tengu_agent_memory_loaded", {
  ...{},
  scope: A.memory,
  isMainLoopAgent: !0,
});
```

### Teammate Events

#### `tengu_teammate_mode_changed`


```typescript
l("tengu_teammate_mode_changed", { mode: T1 });
```

#### `tengu_teammate_*` events


Multiple teammate-related events are tracked:
- Teammate status changes
- Message delivery
- Idle state transitions
- Task assignment/completion

### Task Events

- Task creation
- Task claiming
- Task completion
- Task unassignment

### Hook Events


Hook events fire telemetry for:
- `SubagentStart`: agent_id, agent_type
- `SubagentStop`: agent_id, agent_type, agent_transcript_path
- `TeammateIdle`: teammate_name, team_name
- `TaskCompleted`: task_id, task_subject, task_description, teammate_name, team_name

Each hook has a `matcherMetadata` field:
```typescript
matcherMetadata: { fieldToMatch: "agent_type", values: [] }
```

### Event Schema


Telemetry event schema includes:
```typescript
{
  email: "",
  agent_id: "",
  parent_session_id: "",
  agent_type: "",
  slack: void 0,
  team_name: "",
}
```


Telemetry enrichment:
```typescript
if (w.agentType) $.agent_type = w.agentType;
if (w.teamName) $.team_name = w.teamName;
```

---

## 15. Key Functions Reference Table

### Path Functions

| Function | Purpose |
|----------|---------|
| `$8()` | Get Claude config dir (`~/.claude`) |
| `MW()` | Get teams dir (`~/.claude/teams`) |
| `i06(teamName)` | Get team directory path |
| `hp4(teamName)` | Alias for `i06` |
| `ik(teamName)` | Get task directory path |
| `gkA(name)` | Normalize team/agent name |
| `Ws(agentName, teamName)` | Get inbox file path |
| `Hy1(teamName, taskId)` | Get task file path |

### Config Functions

| Function | Purpose |
|----------|---------|
| `yX(teamName)` | Read team config |
| `FGY(teamName)` | Read team config (permission sync) |
| `E31(teamName)` | Read team config (spawn) |
| `SLY(teamName, config)` | Write team config |
| `cTA(teamName, config)` | Write team config (spawn) |
| `Mm1(teamName, config)` | Write team config (teammate tool) |

### Member Functions

| Function | Purpose |
|----------|---------|
| `lTA(name, teamName)` | Deduplicate member name |
| `dTA(name)` | Clean name (remove @) |
| `PE4(teamName, paneId)` | Remove member by tmux pane |
| `ZE4(teamName, agentInfo)` | Remove member by agent ID/name |

### Agent Lifecycle Functions

| Function | Purpose |
|----------|---------|
| `qLA()` | Main agent execution entry point |
| `EZ()` | Async generator for streaming execution |
| `Y8A()` | Wait for teammates to become idle |
| `sL()` | Agent ID generator |
| `Mt4()` | UUID generator (randomUUID) |
| `Kz()` | Check if running as teammate |
| `iM()` | Check if running as team lead |
| `zy1()` | Check if plan mode required |
| `ct()` | Switch to teammate view |
| `PI()` | Switch to leader view |

### Task Management Functions

| Function | Purpose |
|----------|---------|
| `rO1` | Create task |
| `OU` | Read task |
| `QC` | Update task |
| `zX` | List tasks |
| `T46` | Delete task |
| `J8A` | Claim task |
| `ZG5` | Claim with busy check |
| `In` | Unassign tasks |
| `_8A` | Add dependency |
| `GG5` | Get max task ID |
| `O8A` | Read watermark |

### Messaging Functions

| Function | Purpose |
|----------|---------|
| `X9` | writeToMailbox |
| `ip` | readMailbox |
| `M31` | readUnreadMessages |
| `im1` | markMessagesAsRead |
| `lm1` | markMessageAsReadByIndex |
| `ULY` | Send direct message |
| `gLY` | Send broadcast |
| `pLY` | Send shutdown request |
| `dLY` | Approve shutdown |
| `cLY` | Reject shutdown |
| `lLY` | Approve plan |
| `iLY` | Reject plan |
| `CWY` | formatTeammateMessages |
| `nm1` | createIdleNotification |
| `rm1` | isIdleNotification |
| `sm1` | getLastPeerDmSummary |

### Message Type Checkers

| Function | Purpose |
|----------|---------|
| `Gs()` | isShutdownRequest |
| `TZ()` | isShutdownApproved |
| `gD6()` | isShutdownRejected |
| `UD6()` | isPlanApprovalRequest |
| `dj1()` | isPlanApprovalResponse |
| `hWY()` | isTaskCompletedNotification |
| `iD6()` | isStructuredProtocolMessage |

### Team Context Functions

| Function | Purpose |
|----------|---------|
| `Q3()` | getTeamName |
| `B5()` | getAgentName |
| `_0()` | getAgentId |
| `R$()` | getTeammateColor |
| `lk()` | getTeammateContext |
| `f46` | createTeammateContext |
| `XG5` | setDynamicTeamContext |
| `DG5` | clearDynamicTeamContext |

---

## 16. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Main Session (Team Lead)                        │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    App State Management                           │ │
│  │  - tasks: Map<TaskID, Task>                                       │ │
│  │  - toolPermissionContext: { mode, ... }                           │ │
│  │  - agentDefinitions                                               │ │
│  │  - teamContext: { teamName, leadAgentId, teammates }              │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                         │                                               │
│         ┌───────────────┼───────────────┬───────────────┐               │
│         │               │               │               │               │
│  ┌──────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐ ┌──────▼──────┐        │
│  │ Team Lead   │ │Researcher │ │  Engineer   │ │   Tester    │        │
│  │ (Main Agent)│ │(teammate) │ │ (teammate)  │ │ (teammate)  │        │
│  │             │ │           │ │             │ │             │        │
│  │ agentId: A  │ │agentId: B │ │ agentId: C  │ │ agentId: D  │        │
│  │ status: run │ │status: run│ │ status: idle│ │ status: run │        │
│  │ isIdle: no  │ │isIdle: no │ │ isIdle: yes │ │ isIdle: no  │        │
│  │ mode: deflt │ │mode: plan │ │ mode: deflt │ │ mode: deflt │        │
│  └─────────────┘ └───────────┘ └─────────────┘ └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
          │               │               │               │
          │               │               │               │
          └───────────────┴───────────────┴───────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        File System Storage                              │
│                                                                         │
│  ~/.claude/                                                             │
│    ├── teams/                                                           │
│    │   └── my-team/                                                     │
│    │       ├── config.json     # Team config & member registry          │
│    │       └── inboxes/         # Message inboxes                       │
│    │           ├── team-lead.json                                       │
│    │           ├── researcher.json                                      │
│    │           ├── engineer.json                                        │
│    │           └── tester.json                                          │
│    │                                                                    │
│    └── tasks/                                                           │
│        └── my-team/             # Shared task list                      │
│            ├── 1.json           # Task: "Research auth patterns"        │
│            ├── 2.json           # Task: "Implement OAuth"               │
│            ├── 3.json           # Task: "Write tests"                   │
│            ├── .lock            # Concurrency control                   │
│            └── .highwatermark   # ID reuse prevention                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
          │                                   │
          │                                   │
          ▼                                   ▼
┌─────────────────────────┐       ┌─────────────────────────┐
│  Message Polling        │       │  Task Claiming          │
│                         │       │                         │
│  InboxPoller runs       │       │  Agents read task list  │
│  every N seconds        │       │  Claim available tasks  │
│                         │       │  Check dependencies     │
│  Reads unread messages  │       │  Update ownership       │
│  Categorizes by type    │       │                         │
│  Injects into session   │       │  Auto-unassign on       │
│  Marks as read          │       │  shutdown               │
└─────────────────────────┘       └─────────────────────────┘
```

### Communication Flow

```
Team Lead                     Researcher                   Engineer
    │                             │                            │
    │ SendMessage(broadcast)      │                            │
    │─────────────────────────────┼────────────────────────────│
    │                             │                            │
    │                        writeToMailbox()                  │
    │                             │                            │
    │                     ┌───────▼───────┐            ┌───────▼───────┐
    │                     │ researcher    │            │ engineer      │
    │                     │ .json inbox   │            │ .json inbox   │
    │                     └───────┬───────┘            └───────┬───────┘
    │                             │                            │
    │                     InboxPoller                  InboxPoller
    │                             │                            │
    │                     <teammate_message>           <teammate_message>
    │                             │                            │
    │                             │  SendMessage("engineer")   │
    │                             │───────────────────────────▶│
    │                             │                            │
    │                             │                    writeToMailbox()
    │                             │                            │
    │                             │                    InboxPoller
    │                             │                            │
    │◀────────────────────────────┼────────────────────────────│
    │  Both send idle notifications to team lead               │
    │                             │                            │
```

---

