# Extending Happy for New Agent Integration (OpenCode Example)

**Document Version:** 1.0
**Date:** 2026-01-15
**Scope:** Step-by-step guide for integrating a new AI coding agent (like OpenCode) into the Happy architecture

---

## Executive Summary

This document provides a comprehensive guide for extending Happy to support a new AI coding agent. Using **OpenCode** as the example target, we walk through the architectural patterns, code locations, and integration steps required to add a fourth agent to Happy's existing Claude/Codex/Gemini support.

**Key Finding:** OpenCode is already partially recognized in the codebase (present in ACP protocol enum at `sources/sync/typesRaw.ts:212`), but requires ~7 integration steps to become fully operational.

**Integration Complexity:** ~95% of infrastructure is already shared. Adding OpenCode requires:
- ~200-400 lines of new code (tool-specific views, CLI detection)
- ~50-100 lines of modifications (type unions, conditional logic)
- ~6-8 hours of development time (experienced developer)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Integration Checklist](#integration-checklist)
3. [Step 1: Protocol Definition & Message Normalization](#step-1-protocol-definition--message-normalization)
4. [Step 2: CLI Detection](#step-2-cli-detection)
5. [Step 3: Session Spawning](#step-3-session-spawning)
6. [Step 4: Tool Registration](#step-4-tool-registration)
7. [Step 5: UI Flavor System](#step-5-ui-flavor-system)
8. [Step 6: Profile Compatibility](#step-6-profile-compatibility)
9. [Step 7: Testing & Validation](#step-7-testing--validation)
10. [Common Pitfalls & Debugging](#common-pitfalls--debugging)
11. [Reference: File Locations](#reference-file-locations)

---

## Architecture Overview

### Current Multi-Agent Architecture

Happy supports three agents with ~92% code sharing:

| Agent | Provider | Protocol | CLI Command | Status |
|-------|----------|----------|-------------|--------|
| Claude | Anthropic | Native | `claude` | ✅ Production |
| Codex | OpenAI | ACP | `codex` | ✅ Production |
| Gemini | Google | ACP | `gemini` | ✅ Production (Experimental) |
| **OpenCode** | **TBD** | **ACP** | **`opencode`** | **🚧 To Be Added** |

### Key Architectural Patterns

1. **Single Entry Point**: `machineSpawnNewSession()` handles all agents
2. **Universal CLI Detection**: `useCLIDetection()` hook detects all CLIs in parallel
3. **Protocol Normalization**: `normalizeRawMessage()` converts agent-specific formats to canonical format
4. **Tool Registry**: `knownTools` object maps tool names to UI renderers
5. **Flavor System**: `session.metadata.flavor` field enables runtime UI adaptation

### Shared vs Agent-Specific Code

```
Shared Infrastructure (~15,000+ lines):
├── Session spawning (machineSpawnNewSession)
├── RPC operations (bash, readFile, writeFile, etc.) [100% shared]
├── Sync engine (SyncSocket, SyncSession)
├── Message normalization (normalizeRawMessage)
├── Permission system (PermissionFooter)
└── UI components (Item, Avatar, StatusDot, etc.)

Agent-Specific Code (~800 lines):
├── CLI detection patterns (command -v opencode)
├── Tool views (OpenCodeBashView, OpenCodePatchView, etc.)
├── Protocol message formats (ACP data types)
├── Environment variable transformations
└── Avatar icons (flavor system)
```

---

## Integration Checklist

Use this checklist to track progress when integrating OpenCode:

- [ ] **Step 1: Protocol Definition**
  - [ ] Add `'opencode'` to ACP provider enum (already done at `typesRaw.ts:212`)
  - [ ] Verify message type support (tool-call, tool-result, file-edit, etc.)
  - [ ] Add normalization logic if OpenCode uses non-standard formats

- [ ] **Step 2: CLI Detection**
  - [ ] Add `opencode: boolean | null` to `CLIAvailability` type
  - [ ] Update detection bash command to check for `opencode` CLI
  - [ ] Add CLI status display in new session wizard

- [ ] **Step 3: Session Spawning**
  - [ ] Add `'opencode'` to agent type unions
  - [ ] Update `SpawnSessionOptions` interface
  - [ ] Update daemon spawn command handling

- [ ] **Step 4: Tool Registration**
  - [ ] Identify OpenCode-specific tools (e.g., OpenCodeBash, OpenCodePatch)
  - [ ] Create tool view components
  - [ ] Register tools in `knownTools` registry

- [ ] **Step 5: UI Flavor System**
  - [ ] Add `'opencode'` to flavor type unions
  - [ ] Add OpenCode icon to Avatar component
  - [ ] Update session info display

- [ ] **Step 6: Profile Compatibility**
  - [ ] Add `opencode: boolean` to profile compatibility
  - [ ] Update profile validation logic
  - [ ] Test profile selection workflow

- [ ] **Step 7: Testing**
  - [ ] Test CLI detection on machine with OpenCode installed
  - [ ] Test session spawning with OpenCode profile
  - [ ] Verify tool rendering and permissions
  - [ ] Test message normalization with real OpenCode messages

---

## Step 1: Protocol Definition & Message Normalization

### 1.1 Verify ACP Provider Enum

**File:** `sources/sync/typesRaw.ts`

OpenCode is already listed in the ACP protocol enum:

```typescript
// Line 212: ACP (Agent Communication Protocol)
type: z.literal('acp'),
provider: z.enum(['gemini', 'codex', 'claude', 'opencode']),
```

**Status:** ✅ Already complete

### 1.2 Understand Message Normalization Flow

Happy normalizes agent-specific message formats to a canonical internal format:

```typescript
// OpenCode sends (via ACP):
{ type: 'acp', provider: 'opencode', data: { type: 'tool-call', callId: '123', name: 'bash', input: {...} } }

// Happy normalizes to:
{ role: 'agent', content: [{ type: 'tool-call', id: '123', name: 'bash', input: {...} }] }
```

**Key Function:** `normalizeRawMessage()` at `sources/sync/typesRaw.ts:401`

**Action Required:** Review OpenCode's message format documentation. If it deviates from standard ACP, add custom normalization logic in the `if (raw.content.type === 'acp')` block (lines 643-815).

### 1.3 Message Type Support

The ACP protocol already supports all common message types needed for OpenCode:

| Message Type | Purpose | Supported |
|--------------|---------|-----------|
| `message` | Text output | ✅ |
| `reasoning` | Thinking/reasoning | ✅ |
| `thinking` | Extended thinking | ✅ |
| `tool-call` | Tool invocation | ✅ |
| `tool-result` | Tool response | ✅ |
| `tool-call-result` | Hyphenated variant | ✅ |
| `file-edit` | File modifications | ✅ |
| `terminal-output` | Command output | ✅ |
| `permission-request` | Permission prompts | ✅ |
| `task_started` | Task lifecycle | ✅ |
| `task_complete` | Task completion | ✅ |
| `token_count` | Usage metrics | ✅ |

**Action Required:** None unless OpenCode introduces novel message types. If so, add new Zod schemas to the `data: z.discriminatedUnion('type', [...])` array at line 213.

---

## Step 2: CLI Detection

### 2.1 Update CLI Detection Type

**File:** `sources/hooks/useCLIDetection.ts`

**Current Type (lines 4-11):**
```typescript
interface CLIAvailability {
    claude: boolean | null;
    codex: boolean | null;
    gemini: boolean | null;
    isDetecting: boolean;
    timestamp: number;
    error?: string;
}
```

**Required Change:**
```typescript
interface CLIAvailability {
    claude: boolean | null;
    codex: boolean | null;
    gemini: boolean | null;
    opencode: boolean | null; // ✨ Add this
    isDetecting: boolean;
    timestamp: number;
    error?: string;
}
```

### 2.2 Update Detection Command

**File:** `sources/hooks/useCLIDetection.ts`

**Current Detection (lines 60-65):**
```typescript
const result = await machineBash(
    machineId,
    '(command -v claude >/dev/null 2>&1 && echo "claude:true" || echo "claude:false") && ' +
    '(command -v codex >/dev/null 2>&1 && echo "codex:true" || echo "codex:false") && ' +
    '(command -v gemini >/dev/null 2>&1 && echo "gemini:true" || echo "gemini:false")',
    '/'
);
```

**Required Change:**
```typescript
const result = await machineBash(
    machineId,
    '(command -v claude >/dev/null 2>&1 && echo "claude:true" || echo "claude:false") && ' +
    '(command -v codex >/dev/null 2>&1 && echo "codex:true" || echo "codex:false") && ' +
    '(command -v gemini >/dev/null 2>&1 && echo "gemini:true" || echo "gemini:false") && ' +
    '(command -v opencode >/dev/null 2>&1 && echo "opencode:true" || echo "opencode:false")', // ✨ Add this
    '/'
);
```

### 2.3 Update Parsing Logic

**File:** `sources/hooks/useCLIDetection.ts`

**Current Parsing (lines 72-81):**
```typescript
if (result.success && result.exitCode === 0) {
    const lines = result.stdout.trim().split('\n');
    const cliStatus: { claude?: boolean; codex?: boolean; gemini?: boolean } = {};

    lines.forEach(line => {
        const [cli, status] = line.split(':');
        if (cli && status) {
            cliStatus[cli.trim() as 'claude' | 'codex' | 'gemini'] = status.trim() === 'true';
        }
    });
```

**Required Change:**
```typescript
if (result.success && result.exitCode === 0) {
    const lines = result.stdout.trim().split('\n');
    const cliStatus: { claude?: boolean; codex?: boolean; gemini?: boolean; opencode?: boolean } = {}; // ✨ Add opencode

    lines.forEach(line => {
        const [cli, status] = line.split(':');
        if (cli && status) {
            cliStatus[cli.trim() as 'claude' | 'codex' | 'gemini' | 'opencode'] = status.trim() === 'true'; // ✨ Add opencode
        }
    });
```

### 2.4 Update State Initialization

**File:** `sources/hooks/useCLIDetection.ts`

**Multiple Locations to Update:**

```typescript
// Line 37: Initial state
const [availability, setAvailability] = useState<CLIAvailability>({
    claude: null,
    codex: null,
    gemini: null,
    opencode: null, // ✨ Add this
    isDetecting: false,
    timestamp: 0,
});

// Line 84: Success state
setAvailability({
    claude: cliStatus.claude ?? null,
    codex: cliStatus.codex ?? null,
    gemini: cliStatus.gemini ?? null,
    opencode: cliStatus.opencode ?? null, // ✨ Add this
    isDetecting: false,
    timestamp: Date.now(),
});

// Lines 94, 108: Error states
setAvailability({
    claude: null,
    codex: null,
    gemini: null,
    opencode: null, // ✨ Add this
    isDetecting: false,
    timestamp: 0,
    error: `...`,
});
```

### 2.5 Add CLI Warning Banner (Optional)

**File:** `sources/app/(app)/new/index.tsx`

Follow the pattern for Claude/Codex/Gemini warnings (lines 1293-1507). Example:

```typescript
{selectedMachineId && cliAvailability.opencode === false && !isWarningDismissed('opencode') && !hiddenBanners.opencode && (
    <View style={{
        backgroundColor: theme.colors.box.warning.background,
        borderRadius: 10,
        padding: 12,
        marginBottom: 12,
        borderWidth: 1,
        borderColor: theme.colors.box.warning.border,
    }}>
        <View style={{ flexDirection: 'row', alignItems: 'flex-start', justifyContent: 'space-between', marginBottom: 6 }}>
            <View style={{ flex: 1, flexDirection: 'row', alignItems: 'center', flexWrap: 'wrap', gap: 6, marginRight: 16 }}>
                <Ionicons name="warning" size={16} color={theme.colors.warning} />
                <Text style={{ fontSize: 13, fontWeight: '600', color: theme.colors.text }}>
                    OpenCode CLI Not Detected
                </Text>
                {/* Dismissal buttons - see existing implementation */}
            </View>
        </View>
        <View style={{ flexDirection: 'row', alignItems: 'center', flexWrap: 'wrap', gap: 4 }}>
            <Text style={{ fontSize: 11, color: theme.colors.textSecondary }}>
                Install: npm install -g opencode-cli •
            </Text>
            <Pressable onPress={() => {
                if (Platform.OS === 'web') {
                    window.open('https://opencode.ai/docs/installation', '_blank');
                }
            }}>
                <Text style={{ fontSize: 11, color: theme.colors.textLink }}>
                    View Installation Guide →
                </Text>
            </Pressable>
        </View>
    </View>
)}
```

---

## Step 3: Session Spawning

### 3.1 Update Agent Type Unions

**File:** `sources/sync/ops.ts`

**Current Type (lines 136-141):**
```typescript
export interface SpawnSessionOptions {
    machineId: string;
    directory: string;
    approvedNewDirectoryCreation?: boolean;
    token?: string;
    agent?: 'codex' | 'claude' | 'gemini';
    environmentVariables?: Record<string, string>;
}
```

**Required Change:**
```typescript
export interface SpawnSessionOptions {
    machineId: string;
    directory: string;
    approvedNewDirectoryCreation?: boolean;
    token?: string;
    agent?: 'codex' | 'claude' | 'gemini' | 'opencode'; // ✨ Add opencode
    environmentVariables?: Record<string, string>;
}
```

**Also Update RPC Type (lines 165-171):**
```typescript
const result = await apiSocket.machineRPC<SpawnSessionResult, {
    type: 'spawn-in-directory'
    directory: string
    approvedNewDirectoryCreation?: boolean,
    token?: string,
    agent?: 'codex' | 'claude' | 'gemini' | 'opencode', // ✨ Add opencode
    environmentVariables?: Record<string, string>;
}>(
```

### 3.2 Update New Session Wizard

**File:** `sources/app/(app)/new/index.tsx`

**Multiple Locations to Update:**

```typescript
// Line 313: Agent state initialization
const [agentType, setAgentType] = React.useState<'claude' | 'codex' | 'gemini' | 'opencode'>(() => {
    // ... initialization logic
});

// Line 334: Agent cycling handler
const handleAgentClick = React.useCallback(() => {
    setAgentType(prev => {
        // Cycle: claude -> codex -> gemini -> opencode -> claude
        if (prev === 'claude') return 'codex';
        if (prev === 'codex') return experimentsEnabled ? 'gemini' : 'claude';
        if (prev === 'gemini') return experimentsEnabled ? 'opencode' : 'claude'; // ✨ Add this
        return 'claude'; // opencode -> claude
    });
}, [experimentsEnabled]);
```

### 3.3 Update Profile Validation

**File:** `sources/sync/settings.ts` (inferred location)

Add `opencode: boolean` to the `AIBackendProfile.compatibility` object:

```typescript
export interface AIBackendProfile {
    id: string;
    name: string;
    compatibility: {
        claude: boolean;
        codex: boolean;
        gemini: boolean;
        opencode: boolean; // ✨ Add this
    };
    // ... other fields
}
```

**Update Validation Function:**
```typescript
export function validateProfileForAgent(profile: AIBackendProfile, agent: 'claude' | 'codex' | 'gemini' | 'opencode'): boolean {
    return profile.compatibility[agent] === true;
}
```

### 3.4 Daemon-Side Spawn Handling

**Location:** Happy daemon (separate codebase, not shown in these files)

The daemon must recognize `agent: 'opencode'` and spawn the OpenCode CLI with appropriate flags. Expected command:

```bash
opencode --json --session-id <id> --workspace <path> [environment variables as --env or similar]
```

**Action Required:** Update daemon's `spawn-happy-session` RPC handler to support OpenCode. This is outside the scope of the React Native codebase but is critical for end-to-end functionality.

---

## Step 4: Tool Registration

### 4.1 Identify OpenCode-Specific Tools

**Investigation Steps:**
1. Install OpenCode CLI locally
2. Run `opencode --list-tools` (or equivalent) to see available tools
3. Compare with Claude/Codex/Gemini tools in `sources/components/tools/knownTools.tsx`

**Expected Tools (based on Codex/Gemini patterns):**
- `OpenCodeBash` - Terminal command execution
- `OpenCodeReasoning` - Reasoning/thinking display
- `OpenCodePatch` - File change approval UI
- `OpenCodeDiff` - Diff viewer

### 4.2 Create Tool View Components

**Example:** OpenCodeBashView (based on CodexBashView.tsx)

**File:** `sources/components/tools/views/OpenCodeBashView.tsx`

```typescript
import React from 'react';
import { View, Text } from 'react-native';
import { ToolCall } from '@/sync/typesMessage';
import { Metadata } from '@/sync/storageTypes';
import { StyleSheet, useStyles } from 'react-native-unistyles';
import { Typography } from '@/constants/Typography';
import { resolvePath } from '@/utils/pathUtils';

interface OpenCodeBashViewProps {
    tool: ToolCall;
    metadata: Metadata | null;
}

export function OpenCodeBashView(props: OpenCodeBashViewProps) {
    const { theme } = useStyles();
    const { tool, metadata } = props;

    // Parse command from tool input
    const command = Array.isArray(tool.input?.command)
        ? tool.input.command.join(' ')
        : tool.input?.command || 'Unknown command';

    const parsedCmd = tool.input?.parsed_cmd;
    let operationType: 'read' | 'write' | 'bash' | 'unknown' = 'unknown';

    if (parsedCmd && Array.isArray(parsedCmd) && parsedCmd.length > 0) {
        const firstCmd = parsedCmd[0];
        operationType = firstCmd.type || 'unknown';
    }

    // Special handling for read operations
    if (operationType === 'read' && parsedCmd?.[0]?.name) {
        const filePath = resolvePath(parsedCmd[0].name, metadata);
        return (
            <View style={{ padding: 12 }}>
                <Text style={{ fontSize: 13, color: theme.colors.textSecondary, ...Typography.monospace() }}>
                    Reading: {filePath}
                </Text>
            </View>
        );
    }

    // Default command display
    return (
        <View style={{ padding: 12 }}>
            <Text style={{ fontSize: 13, color: theme.colors.text, ...Typography.monospace() }}>
                {command}
            </Text>
        </View>
    );
}
```

**Repeat for Other Tools:**
- `OpenCodePatchView.tsx` (follow `CodexPatchView.tsx` pattern)
- `OpenCodeDiffView.tsx` (follow `CodexDiffView.tsx` pattern)
- `OpenCodeReasoningView.tsx` (if OpenCode supports reasoning)

### 4.3 Register Tools in knownTools

**File:** `sources/components/tools/knownTools.tsx`

Add entries for each OpenCode tool:

```typescript
'OpenCodeBash': {
    title: t('tools.names.terminal'),
    icon: ICON_TERMINAL,
    minimal: true,
    hideDefaultError: true,
    isMutable: true,
    input: z.object({
        command: z.array(z.string()).describe('The command array to execute'),
        cwd: z.string().optional().describe('Current working directory'),
        parsed_cmd: z.array(z.object({
            type: z.string().describe('Type of parsed command (read, write, bash, etc.)'),
            cmd: z.string().optional().describe('The command string'),
            name: z.string().optional().describe('File name or resource name')
        }).loose()).optional().describe('Parsed command information')
    }).partial().loose(),
    extractSubtitle: (opts: { metadata: Metadata | null, tool: ToolCall }) => {
        // Extract command for subtitle
        if (opts.tool.input?.command && Array.isArray(opts.tool.input.command)) {
            let cmdArray = opts.tool.input.command;
            // Handle shell wrappers
            if (cmdArray.length >= 3 && cmdArray[1] === '-lc') {
                return cmdArray[2];
            }
            return cmdArray.join(' ');
        }
        return null;
    },
    extractDescription: (opts: { metadata: Metadata | null, tool: ToolCall }) => {
        return t('tools.names.terminal');
    }
},
'OpenCodeReasoning': {
    title: (opts: { metadata: Metadata | null, tool: ToolCall }) => {
        if (opts.tool.input?.title && typeof opts.tool.input.title === 'string') {
            return opts.tool.input.title;
        }
        return t('tools.names.reasoning');
    },
    icon: ICON_REASONING,
    minimal: true,
    input: z.object({
        title: z.string().describe('The title of the reasoning')
    }).partial().loose(),
    result: z.object({
        content: z.string().describe('The reasoning content'),
        status: z.enum(['completed', 'in_progress', 'error']).optional()
    }).partial().loose(),
    extractDescription: (opts: { metadata: Metadata | null, tool: ToolCall }) => {
        if (opts.tool.input?.title) {
            return opts.tool.input.title;
        }
        return t('tools.names.reasoning');
    }
},
'OpenCodePatch': {
    title: t('tools.names.applyChanges'),
    icon: ICON_EDIT,
    minimal: true,
    hideDefaultError: true,
    input: z.object({
        auto_approved: z.boolean().optional(),
        changes: z.record(z.string(), z.object({
            add: z.object({ content: z.string() }).optional(),
            modify: z.object({ old_content: z.string(), new_content: z.string() }).optional(),
            delete: z.object({ content: z.string() }).optional()
        }).loose())
    }).partial().loose(),
    extractSubtitle: (opts: { metadata: Metadata | null, tool: ToolCall }) => {
        if (opts.tool.input?.changes && typeof opts.tool.input.changes === 'object') {
            const files = Object.keys(opts.tool.input.changes);
            if (files.length > 0) {
                const path = resolvePath(files[0], opts.metadata);
                const fileName = path.split('/').pop() || path;
                if (files.length > 1) {
                    return t('tools.desc.modifyingMultipleFiles', { file: fileName, count: files.length - 1 });
                }
                return fileName;
            }
        }
        return null;
    },
},
'OpenCodeDiff': {
    title: t('tools.names.viewDiff'),
    icon: ICON_EDIT,
    minimal: false,
    hideDefaultError: true,
    noStatus: true,
    input: z.object({
        unified_diff: z.string().describe('Unified diff content')
    }).partial().loose(),
    result: z.object({
        status: z.literal('completed')
    }).partial().loose(),
    extractSubtitle: (opts: { metadata: Metadata | null, tool: ToolCall }) => {
        if (opts.tool.input?.unified_diff && typeof opts.tool.input.unified_diff === 'string') {
            const diffLines = opts.tool.input.unified_diff.split('\n');
            for (const line of diffLines) {
                if (line.startsWith('+++ b/') || line.startsWith('+++ ')) {
                    const fileName = line.replace(/^\+\+\+ (b\/)?/, '');
                    return fileName.split('/').pop() || fileName;
                }
            }
        }
        return null;
    },
},
```

### 4.4 Update Tool Renderer Mapping

**File:** `sources/components/tools/ToolView.tsx` (inferred location)

Add OpenCode tools to the renderer switch:

```typescript
const renderToolContent = () => {
    switch (tool.name) {
        case 'OpenCodeBash':
            return <OpenCodeBashView tool={tool} metadata={metadata} />;
        case 'OpenCodeReasoning':
            return <OpenCodeReasoningView tool={tool} metadata={metadata} />;
        case 'OpenCodePatch':
            return <OpenCodePatchView tool={tool} metadata={metadata} />;
        case 'OpenCodeDiff':
            return <OpenCodeDiffView tool={tool} metadata={metadata} />;
        // ... other tools
        default:
            return <DefaultToolView tool={tool} />;
    }
};
```

---

## Step 5: UI Flavor System

### 5.1 Update Flavor Type Definition

**File:** `sources/sync/storageTypes.ts`

**Current Definition (line 24):**
```typescript
flavor: z.string().nullish() // Session flavor/variant identifier
```

**No changes needed** - the field is already typed as `string | null | undefined`, so `'opencode'` is already valid.

### 5.2 Add OpenCode Avatar Icon

**File:** `sources/components/Avatar.tsx`

**Current flavorIcons (lines 21-30):**
```typescript
const flavorIcons = {
    claude: require('@/assets/images/claude-logo.png'),
    codex: require('@/assets/images/openai-logo.png'),
    gemini: require('@/assets/images/gemini-logo.png'),
    gpt: require('@/assets/images/openai-logo.png'), // Alias for codex
    openai: require('@/assets/images/openai-logo.png'), // Alias for codex
};
```

**Required Change:**
```typescript
const flavorIcons = {
    claude: require('@/assets/images/claude-logo.png'),
    codex: require('@/assets/images/openai-logo.png'),
    gemini: require('@/assets/images/gemini-logo.png'),
    opencode: require('@/assets/images/opencode-logo.png'), // ✨ Add this
    gpt: require('@/assets/images/openai-logo.png'),
    openai: require('@/assets/images/openai-logo.png'),
};
```

**Action Required:** Obtain OpenCode's brand logo and add it to `sources/assets/images/opencode-logo.png`. Recommended dimensions: 48x48px PNG with transparency.

### 5.3 Update Session Info Display

**File:** `sources/app/(app)/session/[id]/info.tsx`

**Current Flavor Display (lines 313-317):**
```typescript
const flavor = session.metadata.flavor || 'claude';
if (flavor === 'claude') return 'Claude';
if (flavor === 'gpt' || flavor === 'openai') return 'Codex';
if (flavor === 'gemini') return 'Gemini';
return flavor;
```

**Required Change:**
```typescript
const flavor = session.metadata.flavor || 'claude';
if (flavor === 'claude') return 'Claude';
if (flavor === 'gpt' || flavor === 'openai') return 'Codex';
if (flavor === 'gemini') return 'Gemini';
if (flavor === 'opencode') return 'OpenCode'; // ✨ Add this
return flavor; // Fallback for unknown flavors
```

### 5.4 Update Permission Footer Logic (Optional)

**File:** `sources/components/tools/PermissionFooter.tsx`

**Current Codex Detection (line 31):**
```typescript
const isCodex = metadata?.flavor === 'codex' || toolName.startsWith('Codex');
```

If OpenCode uses the same permission modes as Codex (default, read-only, safe-yolo, yolo), update:

```typescript
const isCodexOrOpenCode =
    metadata?.flavor === 'codex' ||
    metadata?.flavor === 'opencode' ||
    toolName.startsWith('Codex') ||
    toolName.startsWith('OpenCode');
```

Otherwise, add separate OpenCode permission handling logic.

### 5.5 Update AgentInput Component

**File:** `sources/components/AgentInput.tsx`

**Current Agent Detection (lines 304-305):**
```typescript
const isCodex = props.metadata?.flavor === 'codex' || props.agentType === 'codex';
const isGemini = props.metadata?.flavor === 'gemini' || props.agentType === 'gemini';
```

**Required Change:**
```typescript
const isCodex = props.metadata?.flavor === 'codex' || props.agentType === 'codex';
const isGemini = props.metadata?.flavor === 'gemini' || props.agentType === 'gemini';
const isOpenCode = props.metadata?.flavor === 'opencode' || props.agentType === 'opencode'; // ✨ Add this
```

Then update permission mode rendering to include OpenCode (if it shares Codex modes):

```typescript
if (isCodex || isOpenCode) {
    // Render Codex/OpenCode-style permission modes (default, read-only, safe-yolo, yolo)
} else {
    // Render Claude-style permission modes (default, acceptEdits, plan, bypassPermissions)
}
```

---

## Step 6: Profile Compatibility

### 6.1 Update Profile Interface

**File:** `sources/sync/settings.ts` (inferred from usage patterns)

**Current Interface:**
```typescript
export interface AIBackendProfile {
    id: string;
    name: string;
    compatibility: {
        claude: boolean;
        codex: boolean;
        gemini: boolean;
    };
    anthropicConfig?: {...};
    openaiConfig?: {...};
    environmentVariables?: Array<{name: string; value: string}>;
    // ... other fields
}
```

**Required Change:**
```typescript
export interface AIBackendProfile {
    id: string;
    name: string;
    compatibility: {
        claude: boolean;
        codex: boolean;
        gemini: boolean;
        opencode: boolean; // ✨ Add this
    };
    anthropicConfig?: {...};
    openaiConfig?: {...};
    opencodeConfig?: { // ✨ Add OpenCode-specific config
        model?: string;
        apiKey?: string;
        baseUrl?: string;
    };
    environmentVariables?: Array<{name: string; value: string}>;
    // ... other fields
}
```

### 6.2 Update Built-In Profiles

**File:** `sources/sync/profileUtils.ts` (inferred location)

Add OpenCode to default profiles:

```typescript
export const DEFAULT_PROFILES = [
    {
        id: 'anthropic',
        name: 'Anthropic (Claude)',
        compatibility: { claude: true, codex: false, gemini: false, opencode: false },
        // ...
    },
    {
        id: 'openai',
        name: 'OpenAI (Codex)',
        compatibility: { claude: false, codex: true, gemini: false, opencode: false },
        // ...
    },
    {
        id: 'google',
        name: 'Google (Gemini)',
        compatibility: { claude: false, codex: false, gemini: true, opencode: false },
        // ...
    },
    {
        id: 'opencode-default', // ✨ Add OpenCode profile
        name: 'OpenCode',
        compatibility: { claude: false, codex: false, gemini: false, opencode: true },
        opencodeConfig: {
            model: 'opencode-default',
        },
        environmentVariables: [
            { name: 'OPENCODE_API_KEY', value: '${OPENCODE_API_KEY}' },
            { name: 'OPENCODE_BASE_URL', value: 'https://api.opencode.ai' },
        ],
        isBuiltIn: true,
        // ...
    },
];
```

### 6.3 Update Profile Validation

**File:** `sources/sync/settings.ts`

**Current Validation:**
```typescript
export function validateProfileForAgent(profile: AIBackendProfile, agent: 'claude' | 'codex' | 'gemini'): boolean {
    return profile.compatibility[agent] === true;
}
```

**Required Change:**
```typescript
export function validateProfileForAgent(
    profile: AIBackendProfile,
    agent: 'claude' | 'codex' | 'gemini' | 'opencode' // ✨ Add opencode
): boolean {
    return profile.compatibility[agent] === true;
}
```

### 6.4 Update Environment Variable Transformation

**File:** `sources/app/(app)/new/index.tsx`

**Current Function (lines 61-65):**
```typescript
const transformProfileToEnvironmentVars = (profile: AIBackendProfile, agentType: 'claude' | 'codex' | 'gemini' = 'claude') => {
    return getProfileEnvironmentVariables(profile);
};
```

**Update Type:**
```typescript
const transformProfileToEnvironmentVars = (
    profile: AIBackendProfile,
    agentType: 'claude' | 'codex' | 'gemini' | 'opencode' = 'claude' // ✨ Add opencode
) => {
    return getProfileEnvironmentVariables(profile);
};
```

Then in `getProfileEnvironmentVariables()` function (inferred location in `sources/sync/settings.ts`), add OpenCode config handling:

```typescript
export function getProfileEnvironmentVariables(profile: AIBackendProfile): Record<string, string> {
    const vars: Record<string, string> = {};

    // Anthropic (Claude)
    if (profile.anthropicConfig) {
        if (profile.anthropicConfig.model) vars['ANTHROPIC_MODEL'] = profile.anthropicConfig.model;
        if (profile.anthropicConfig.apiKey) vars['ANTHROPIC_AUTH_TOKEN'] = profile.anthropicConfig.apiKey;
        // ... other Anthropic vars
    }

    // OpenAI (Codex)
    if (profile.openaiConfig) {
        if (profile.openaiConfig.model) vars['OPENAI_MODEL'] = profile.openaiConfig.model;
        if (profile.openaiConfig.apiKey) vars['OPENAI_API_KEY'] = profile.openaiConfig.apiKey;
        // ... other OpenAI vars
    }

    // ✨ OpenCode
    if (profile.opencodeConfig) {
        if (profile.opencodeConfig.model) vars['OPENCODE_MODEL'] = profile.opencodeConfig.model;
        if (profile.opencodeConfig.apiKey) vars['OPENCODE_API_KEY'] = profile.opencodeConfig.apiKey;
        if (profile.opencodeConfig.baseUrl) vars['OPENCODE_BASE_URL'] = profile.opencodeConfig.baseUrl;
    }

    // Custom environment variables from profile
    if (profile.environmentVariables) {
        profile.environmentVariables.forEach(ev => {
            vars[ev.name] = ev.value;
        });
    }

    return vars;
}
```

---

## Step 7: Testing & Validation

### 7.1 Pre-Integration Testing Checklist

Before testing OpenCode integration, ensure:

- [ ] OpenCode CLI is installed on test machine (`npm install -g opencode-cli`)
- [ ] Happy daemon is updated to recognize `agent: 'opencode'` parameter
- [ ] OpenCode API credentials are configured (if required)
- [ ] Test workspace directory exists with sample code

### 7.2 CLI Detection Testing

**Test Steps:**

1. Navigate to New Session screen in Happy app
2. Select test machine
3. Wait for CLI detection to complete
4. Verify OpenCode status indicator shows:
   - ✅ Green checkmark if `opencode` CLI is found
   - ❌ Red X if not found
5. If not found, verify warning banner appears with installation instructions

**Expected Output:**

```
Machine: test-machine
● online  ✓ claude  ✓ codex  ✓ gemini  ✓ opencode
```

**Debugging:**

```bash
# On test machine, manually run detection command:
command -v opencode
# Should print: /path/to/opencode (or nothing if not installed)

# Check Happy daemon logs for detection results
tail -f ~/.happy/daemon.log | grep -i "cli-detection"
```

### 7.3 Profile Selection Testing

**Test Steps:**

1. In New Session wizard, expand profile list
2. Create new custom profile with OpenCode compatibility enabled
3. Verify profile validation:
   - Profile shows "OpenCode CLI" indicator
   - Profile is selectable when `opencode` CLI is detected
   - Profile shows warning if OpenCode CLI not detected
4. Select OpenCode profile
5. Verify agent type auto-selects to OpenCode (if profile is OpenCode-exclusive)

**Expected Behavior:**

- OpenCode-compatible profiles are highlighted when OpenCode is selected
- Profiles incompatible with OpenCode are grayed out
- CLI detection warnings appear for OpenCode-only profiles on machines without OpenCode

### 7.4 Session Spawning Testing

**Test Steps:**

1. Select OpenCode profile
2. Choose test machine and directory
3. Set permission mode (e.g., "Default")
4. Click "Create Session"
5. Monitor session creation:
   - Loading indicator appears
   - No timeout errors (wait up to 30 seconds)
   - Session transitions to active state
6. Verify session metadata:
   ```typescript
   session.metadata.flavor === 'opencode'
   ```

**Expected Behavior:**

- Session spawns successfully
- OpenCode CLI process starts on remote machine
- Session appears in Sessions List with OpenCode avatar icon
- No "Failed to spawn session" errors

**Debugging:**

```bash
# On test machine, check for OpenCode process:
ps aux | grep opencode

# Check daemon logs for spawn command:
tail -f ~/.happy/daemon.log | grep -i "spawn-happy-session"

# Verify environment variables passed to OpenCode:
cat /proc/<opencode-pid>/environ | tr '\0' '\n' | grep OPENCODE
```

### 7.5 Message Normalization Testing

**Test Steps:**

1. In active OpenCode session, send a message: `"List files in current directory"`
2. Observe tool calls from OpenCode (e.g., `OpenCodeBash`)
3. Verify tools render correctly in UI
4. Check browser console for normalization errors:
   ```
   [normalizeRawMessage] Validation error: ...
   ```
5. Send multiple message types:
   - Simple text response
   - Tool call (bash command)
   - Tool result (command output)
   - Reasoning/thinking (if supported)

**Expected Behavior:**

- All OpenCode messages normalize without errors
- Tools display with correct icons and formatting
- Tool results show command output
- No `=== VALIDATION ERROR ===` logs in console

**Debugging:**

```typescript
// Add temporary logging in normalizeRawMessage (typesRaw.ts:401)
export function normalizeRawMessage(id: string, localId: string | null, createdAt: number, raw: RawRecord): NormalizedMessage | null {
    console.log('[normalizeRawMessage] Input:', JSON.stringify(raw, null, 2)); // ✨ Add this

    let parsed = rawRecordSchema.safeParse(raw);
    if (!parsed.success) {
        console.error('=== VALIDATION ERROR ===');
        console.error('Zod issues:', JSON.stringify(parsed.error.issues, null, 2));
        // ...
    }
    // ... rest of function
}
```

### 7.6 Tool Rendering Testing

**Test Steps:**

1. Send message triggering each OpenCode tool:
   - `"Show me app.py"` → OpenCodeBash (read operation)
   - `"Run npm test"` → OpenCodeBash (command)
   - `"Change greeting to 'Hello World'"` → OpenCodePatch
   - `"Show me the diff"` → OpenCodeDiff
2. Verify each tool:
   - Uses correct icon (terminal, edit, etc.)
   - Shows correct title/subtitle
   - Displays input/result properly
   - Permission footer works (if applicable)

**Expected Behavior:**

- OpenCode tools render with same quality as Claude/Codex tools
- File paths are clickable and resolve correctly
- Diffs display with syntax highlighting
- Commands show full output (not truncated)

**Debugging:**

```typescript
// Check if tool is registered (browser console):
import { knownTools } from '@/components/tools/knownTools';
console.log('OpenCodeBash registered:', 'OpenCodeBash' in knownTools);
console.log('Tool config:', knownTools['OpenCodeBash']);
```

### 7.7 Permission Mode Testing

**Test Steps:**

1. Create sessions with each permission mode:
   - Default (prompt for permissions)
   - Read-only (if OpenCode supports it)
   - Safe YOLO (if OpenCode supports it)
   - YOLO (if OpenCode supports it)
2. For each mode, send message requiring tool use
3. Verify permission behavior matches expected mode

**Expected Behavior:**

- **Default:** Permission prompt appears before tool execution
- **Read-only:** Only read operations allowed, writes blocked
- **Safe YOLO:** Workspace writes auto-approved, dangerous operations require approval
- **YOLO:** All operations auto-approved

### 7.8 End-to-End Integration Test

**Complete Workflow:**

1. ✅ CLI Detection: OpenCode detected on machine
2. ✅ Profile Selection: Select "OpenCode" built-in profile
3. ✅ Session Creation: Spawn OpenCode session in `/path/to/test/project`
4. ✅ Message Exchange: Send message "Add docstring to main function"
5. ✅ Tool Execution: OpenCode uses tools (read file, edit file, show diff)
6. ✅ Permission Approval: Approve file edit
7. ✅ Verification: Check that file was modified correctly
8. ✅ Session Management: Archive session, verify it appears in Sessions List

**Success Criteria:**

- All steps complete without errors
- OpenCode session behaves identically to Claude/Codex sessions
- UI shows OpenCode branding (avatar icon, flavor)
- Performance is acceptable (~2-5 second response time)

---

## Common Pitfalls & Debugging

### Pitfall 1: Flavor Not Set on Session

**Symptom:** Session appears with Claude avatar instead of OpenCode avatar.

**Cause:** Daemon not setting `flavor: 'opencode'` in session metadata.

**Fix:**

Check daemon's session spawn code:

```javascript
// Daemon code (pseudo-code)
const metadata = {
    path: options.directory,
    host: os.hostname(),
    flavor: options.agent, // ✨ Must map agent to flavor
    // ...
};
```

### Pitfall 2: Tool Names Don't Match

**Symptom:** OpenCode tools show as "Unknown tool" with no icon.

**Cause:** OpenCode CLI sends tool names that don't match `knownTools` registry.

**Fix:**

1. Inspect raw tool call:
   ```typescript
   console.log('Tool name from OpenCode:', tool.name);
   ```
2. Update tool name in OpenCode CLI (ideal), OR
3. Add aliases in `knownTools`:
   ```typescript
   'opencode_bash': knownTools['OpenCodeBash'], // Alias
   ```

### Pitfall 3: Protocol Mismatch

**Symptom:** Validation errors in `normalizeRawMessage`:

```
=== VALIDATION ERROR ===
Zod issues: [{"code":"invalid_union","message":"Invalid discriminator value..."}]
```

**Cause:** OpenCode sends messages in a format not recognized by ACP schema.

**Fix:**

1. Compare OpenCode message structure with Codex/Gemini
2. If OpenCode uses different field names (e.g., `toolId` instead of `callId`), add transformation:
   ```typescript
   // In normalizeRawMessage, before validation:
   if (raw.content?.provider === 'opencode') {
       // Transform OpenCode-specific format to ACP standard
       if (raw.content.data?.toolId) {
           raw.content.data.callId = raw.content.data.toolId;
           delete raw.content.data.toolId;
       }
   }
   ```

### Pitfall 4: Environment Variables Not Passed

**Symptom:** OpenCode session fails to start with "API key not found" error.

**Cause:** Profile environment variables not reaching OpenCode CLI.

**Debugging:**

```bash
# On test machine, check OpenCode process environment:
cat /proc/<opencode-pid>/environ | tr '\0' '\n' | grep OPENCODE

# Should show:
OPENCODE_API_KEY=sk-...
OPENCODE_BASE_URL=https://api.opencode.ai
OPENCODE_MODEL=opencode-default
```

**Fix:**

1. Verify profile has `environmentVariables` array populated
2. Check `getProfileEnvironmentVariables()` includes OpenCode config
3. Confirm daemon passes environment variables to `spawn()` call

### Pitfall 5: CLI Detection False Positive

**Symptom:** OpenCode shows as "detected" but session spawn fails.

**Cause:** `command -v opencode` returns a path, but the executable is broken or incompatible.

**Fix:**

1. Update detection to test CLI version:
   ```bash
   # New detection command:
   (command -v opencode >/dev/null 2>&1 && opencode --version >/dev/null 2>&1 && echo "opencode:true" || echo "opencode:false")
   ```
2. Or, accept false positives and let spawn error provide user feedback

### Debugging Resources

**Browser Console:**
- Check `localStorage` for settings: `localStorage.getItem('happy:settings')`
- Inspect session metadata: `storage.getState().sessions['session-id'].metadata`

**Happy Daemon Logs:**
```bash
# Linux/macOS:
tail -f ~/.happy/daemon.log

# Grep for OpenCode-related entries:
grep -i opencode ~/.happy/daemon.log
```

**Network Tab (DevTools):**
- Monitor WebSocket messages for session spawn
- Look for `machine-rpc` calls with `agent: 'opencode'`

---

## Reference: File Locations

### Files Requiring Modification

| File | Lines to Modify | Purpose |
|------|----------------|---------|
| `sources/sync/typesRaw.ts` | 212 | Add 'opencode' to ACP provider enum (✅ already done) |
| `sources/hooks/useCLIDetection.ts` | 4-11, 37, 60-65, 72-81, 84, 94, 108 | CLI detection for OpenCode |
| `sources/sync/ops.ts` | 136-141, 165-171 | Session spawning types |
| `sources/app/(app)/new/index.tsx` | 313, 334, 370, 460-479 | New session wizard |
| `sources/sync/settings.ts` | Interface definition | Profile compatibility |
| `sources/components/tools/knownTools.tsx` | New entries | Tool registration |
| `sources/components/Avatar.tsx` | 21-30 | Flavor icons |
| `sources/app/(app)/session/[id]/info.tsx` | 313-317 | Flavor display |
| `sources/components/AgentInput.tsx` | 304-305 | Agent detection |

### Files Requiring Creation

| File | Purpose |
|------|---------|
| `sources/components/tools/views/OpenCodeBashView.tsx` | Bash command rendering |
| `sources/components/tools/views/OpenCodePatchView.tsx` | File change approval UI |
| `sources/components/tools/views/OpenCodeDiffView.tsx` | Diff viewer |
| `sources/components/tools/views/OpenCodeReasoningView.tsx` | Reasoning display (if applicable) |
| `sources/assets/images/opencode-logo.png` | OpenCode brand icon |

### External Dependencies

| Component | Location | Required Changes |
|-----------|----------|------------------|
| Happy Daemon | Separate codebase | Recognize `agent: 'opencode'`, spawn OpenCode CLI |
| OpenCode CLI | External package | Must support JSON output, session IDs, environment variables |

---

## Conclusion

Integrating OpenCode (or any new agent) into Happy requires touching ~15-20 files with ~250-500 lines of new code. The architecture's high degree of code sharing (92%) means most functionality works automatically once the core integration points are addressed.

**Estimated Timeline:**

- **Experienced Developer:** 6-8 hours
- **New to Happy Codebase:** 12-16 hours

**Critical Success Factors:**

1. OpenCode CLI must support JSON-based messaging (ACP protocol preferred)
2. Happy daemon must be updated to spawn OpenCode CLI
3. Tool names must be consistent and documented
4. Environment variable handling must match expected format

**Next Steps:**

1. Install OpenCode CLI locally and test basic functionality
2. Document OpenCode's tool names and message formats
3. Update Happy daemon to support OpenCode spawning
4. Follow this guide step-by-step, testing at each integration point
5. Submit PR with comprehensive tests and documentation

---

**Questions or Issues?** Refer to the Happy codebase documentation or create an issue at the Happy GitHub repository.
