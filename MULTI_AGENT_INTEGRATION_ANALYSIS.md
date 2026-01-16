# Multi-Agent Integration Analysis: Codex & Gemini Support

**Repository:** Happy - Mobile and Web Client for Claude Code & Codex
**Analysis Date:** January 15, 2026
**Purpose:** Evaluate how Happy integrates with multiple AI code agents (Claude, Codex, Gemini) via ACP

---

## Executive Summary

The Happy repository demonstrates **production-grade multi-agent support** through a custom **ACP (Agent Communication Protocol)** implementation. This protocol-based architecture allows Happy to work seamlessly with:

1. **Claude** (Anthropic) - via Claude Code CLI
2. **Codex** (OpenAI) - via Codex CLI
3. **Gemini** (Google) - via ACP protocol

**Integration Type:** Hybrid (Documentation-based for Claude + Protocol-based for Codex/Gemini)
**Integration Maturity:** Advanced (Production) ⭐⭐⭐⭐⭐
**Architecture:** Unified protocol with agent-specific UI components

---

## 1. Supported AI Code Agents

### 1.1 Agent Overview

| Agent | Provider | Integration Method | Status |
|-------|----------|-------------------|--------|
| **Claude Code** | Anthropic | Documentation (CLAUDE.md) + Direct CLI | Production |
| **Codex** | OpenAI | ACP Protocol + Custom UI | Production |
| **Gemini** | Google | ACP Protocol + Custom UI | Experimental |

### 1.2 Agent Communication Protocol (ACP)

**Location:** `sources/sync/typesRaw.ts:210-271`

```typescript
{
    type: z.literal('acp'),
    provider: z.enum(['gemini', 'codex', 'claude', 'opencode']),
    data: z.discriminatedUnion('type', [...])
}
```

**Supported Providers:**
- `codex` - **OpenAI Codex CLI** (primary ACP user)
- `gemini` - **Google Gemini** (experimental)
- `claude` - Anthropic Claude (also uses native format)
- `opencode` - Reserved for future providers

---

## 2. Codex Integration (OpenAI)

### 2.1 What is Codex?

From OpenAI's documentation (2025):
- **Codex CLI** - OpenAI's terminal-based coding agent
- Built in Rust for speed and efficiency
- Included with ChatGPT Plus, Pro, Business, Edu, Enterprise
- Can read, change, and run code on your machine
- Full-screen terminal UI with repository awareness

### 2.2 Codex-Specific Components

#### CodexBashView (`sources/components/tools/views/CodexBashView.tsx`)

**Handles:** Codex's bash/shell commands with parsed command metadata

```typescript
{
    command: string[],              // Command array
    cwd: string,                    // Working directory
    parsed_cmd: [{                  // Parsed command info
        type: 'read' | 'write' | 'bash',
        name: string,               // File name
        cmd: string                 // Command string
    }]
}
```

**Features:**
- Detects operation type (read/write/bash)
- Shows file name for file operations
- Displays command with proper icons
- Read operations show "Reading file.txt"
- Write operations show "Writing file.txt"
- Bash operations show terminal command

#### CodexPatchView (`sources/components/tools/views/CodexPatchView.tsx`)

**Handles:** Codex's file patching tool

```typescript
{
    auto_approved: boolean,
    changes: {
        [filePath]: {
            add?: { content: string },
            modify?: { old_content: string, new_content: string },
            delete?: { content: string }
        }
    }
}
```

**Features:**
- Lists all files being modified
- Single file: shows inline
- Multiple files: shows as list with count
- File icons for visual clarity

#### CodexDiffView (`sources/components/tools/views/CodexDiffView.tsx`)

**Handles:** Codex's unified diff display

**Features:**
- Parses unified diff format (git-style)
- Extracts old/new content
- Shows file name header
- Displays diff with syntax highlighting
- Supports line numbers toggle

### 2.3 Codex Tool Definitions

**Location:** `sources/components/tools/knownTools.tsx:440-518, 687-847`

Codex-specific tools:

```typescript
export const knownTools = {
    'CodexBash': {
        title: /* File name or 'Terminal' */,
        icon: ICON_TERMINAL,
        minimal: true,
        input: z.object({
            command: z.array(z.string()),
            cwd: z.string().optional(),
            parsed_cmd: z.array(z.object({
                type: z.string(),
                cmd: z.string().optional(),
                name: z.string().optional()
            }))
        })
    },
    'CodexReasoning': {
        title: /* Reasoning title */,
        icon: ICON_REASONING,
        minimal: true,
        input: z.object({
            title: z.string()
        })
    },
    'CodexPatch': {
        title: 'Apply Changes',
        icon: ICON_EDIT,
        minimal: true,
        hideDefaultError: true,
        input: z.object({
            auto_approved: z.boolean().optional(),
            changes: z.record(...)
        })
    },
    'CodexDiff': {
        title: 'View Diff',
        icon: ICON_EDIT,
        minimal: false,  // Always show full diff
        noStatus: true,
        input: z.object({
            unified_diff: z.string()
        })
    }
}
```

**Tool Count:** 4 Codex-specific tool handlers

---

## 3. Gemini Integration (Google)

### 3.1 What is Gemini?

Google's Gemini code agent integration via ACP protocol.

### 3.2 Gemini-Specific Components

#### GeminiEditView (`sources/components/tools/views/GeminiEditView.tsx`)

**Handles:** Gemini's `edit` tool with nested data structures

```typescript
/**
 * Extract edit content from Gemini's nested input format.
 *
 * Gemini sends data in nested structure:
 * - tool.input.toolCall.content[0]
 * - tool.input.input[0]
 * - tool.input (direct fields)
 */
function extractEditContent(input: any): {
    oldText: string;  // Maps to Claude's old_string
    newText: string;  // Maps to Claude's new_string
    path: string;
}
```

**Features:**
- Handles 3 different nested formats from Gemini
- Maps `oldText`/`newText` to standard format
- Renders diffs with line numbers
- Integrates with theme system

#### GeminiExecuteView (`sources/components/tools/views/GeminiExecuteView.tsx`)

**Handles:** Gemini's `execute` tool (shell commands)

```typescript
/**
 * Extract execute command info from Gemini's nested input format.
 * Title format: "rm file.txt [current working directory /path] (description)"
 */
function extractExecuteInfo(input: any): {
    command: string;
    description: string;
    cwd: string;
}
```

**Features:**
- Extracts command, working directory, description
- Parses Gemini's title format with regex
- Shows current working directory with 📁 icon
- Displays command description in italics

### 3.3 Gemini Tool Definitions

**Location:** `sources/components/tools/knownTools.tsx:543-884`

Gemini-specific tools:

```typescript
export const knownTools = {
    // Gemini internal tools (minimal/hidden)
    'search': { minimal: true },
    'read': { /* Lowercase variant */ },
    'edit': { /* Gemini edit format */ },
    'shell': { minimal: true },
    'execute': { /* Gemini execute */ },
    'think': { /* Gemini thinking */ },

    // Gemini-prefixed tools
    'GeminiReasoning': { /* Reasoning steps */ },
    'GeminiBash': { /* Terminal commands */ },
    'GeminiPatch': { /* File patching */ },
    'GeminiDiff': { /* Diff display */ }
}
```

**Tool Count:** 10+ Gemini-specific tool handlers

### 3.4 Auto-Hidden Tools

**Location:** `sources/components/tools/ToolView.tsx:54-60`

```typescript
// For Gemini: unknown tools should be rendered as minimal (hidden)
// This prevents showing raw INPUT/OUTPUT for internal Gemini tools
// that we haven't explicitly added to knownTools
const isGemini = props.metadata?.flavor === 'gemini';
if (!knownTool && isGemini) {
    minimal = true;
}
```

**Purpose:** Automatically hide unknown Gemini internal tools to avoid UI clutter

---

## 4. Protocol Translation Layer

### 4.1 Message Normalization

**Location:** `sources/sync/typesRaw.ts:642-815`

ACP messages are normalized to internal format:

```typescript
// ACP (Agent Communication Protocol)
if (raw.content.type === 'acp') {
    if (raw.content.data.type === 'message') {
        // Normalize to standard text message
        return {
            role: 'agent',
            content: [{
                type: 'text',
                text: raw.content.data.message
            }]
        };
    }

    if (raw.content.data.type === 'tool-call') {
        // Normalize Codex/Gemini tool calls
        return {
            role: 'agent',
            content: [{
                type: 'tool-call',
                id: raw.content.data.callId,
                name: raw.content.data.name,
                input: raw.content.data.input
            }]
        };
    }

    // ... file-edit, terminal-output, thinking, etc.
}
```

**Translation Mappings:**

| ACP Type | Normalized Type | Used By | Description |
|----------|----------------|---------|-------------|
| `message` | `text` | Codex, Gemini | Regular text messages |
| `reasoning` | `text` | Codex, Gemini | Reasoning/thinking steps |
| `thinking` | `thinking` | Codex, Gemini | Extended thinking content |
| `tool-call` | `tool-call` | Codex, Gemini | Tool invocations |
| `tool-result` | `tool-result` | Codex, Gemini | Tool results |
| `file-edit` | `tool-call` | Codex, Gemini | File edit operations |
| `terminal-output` | `tool-result` | Codex, Gemini | Terminal command output |
| `permission-request` | `tool-call` | Codex, Gemini | Permission dialogs |
| `task_started` | (skipped) | Codex | Task lifecycle event |
| `task_complete` | (skipped) | Codex | Task completion event |

### 4.2 Hyphenated Format Support

**Location:** `sources/sync/typesRaw.ts:87-169`

Codex and Gemini use **hyphenated** format instead of Claude's underscore format:

```typescript
/**
 * Hyphenated tool-call format from Codex/Gemini agents
 * Transforms to canonical tool_use format during validation
 */
const rawHyphenatedToolCallSchema = z.object({
    type: z.literal('tool-call'),  // Codex/Gemini format
    callId: z.string(),
    name: z.string(),
    input: z.any(),
}).passthrough();

// Transforms to:
{
    type: 'tool_use',  // Claude format (canonical)
    id: callId,
    name: name,
    input: input
}
```

**Format Comparison:**

| Format | Used By | Example Types |
|--------|---------|--------------|
| Hyphenated | Codex, Gemini | `tool-call`, `tool-call-result` |
| Underscore | Claude | `tool_use`, `tool_result` |
| Normalized | Internal | `tool-call` → `tool_use` |

**Benefit:** Single codebase handles both formats seamlessly

---

## 5. Session Flavor Detection

### 5.1 Metadata Flavor Field

**Location:** `sources/sync/storageTypes.ts:24`

```typescript
export const MetadataSchema = z.object({
    path: z.string(),
    host: z.string(),
    // ... other fields ...
    flavor: z.string().nullish()  // Session flavor/variant identifier
});
```

**Flavor Values:**
- `null` or `undefined` - **Claude** (default)
- `"codex"` - **OpenAI Codex**
- `"gemini"` - **Google Gemini**

### 5.2 Flavor-Based UI Rendering

**Location:** `sources/components/tools/ToolView.tsx:57-60`

```typescript
const isGemini = props.metadata?.flavor === 'gemini';
if (!knownTool && isGemini) {
    minimal = true;  // Hide unknown Gemini tools
}
```

**Effect:** UI behavior adapts based on agent flavor

---

## 6. Model Selection

### 6.1 Model Modes by Agent

**Location:** `sources/sync/storageTypes.ts:73`

```typescript
export interface Session {
    // ... other fields ...
    modelMode?:
        | 'default'
        | 'gemini-2.5-pro'
        | 'gemini-2.5-flash'
        | 'gemini-2.5-flash-lite'
        | null;
}
```

**Note:** Gemini models are currently the only ones with explicit model selection in the mobile app.

### 6.2 Model Configuration Changes

**Changelog Entry (Version 5):**
```markdown
## Version 5 - 2025-12-22
- Removed model configurations from agents. We were not able to keep up
  with the models so for now we are removing the configuration from the
  mobile app. You can still configure it through your CLIs, happy will
  simply use defaults.
```

**Current State:** Model selection moved to CLI configuration

---

## 7. Tool System Compatibility

### 7.1 Tool Mappings Across Agents

| Function | Claude Tool | Codex Tool | Gemini Tool | Notes |
|----------|-------------|------------|-------------|-------|
| Read file | `Read` | `CodexBash` (parsed) | `read` | Codex uses bash with parsed_cmd |
| Edit file | `Edit` | `CodexPatch` | `edit` | Different data structures |
| Shell command | `Bash` | `CodexBash` | `execute` | Codex has parsed metadata |
| Show diff | N/A | `CodexDiff` | `GeminiDiff` | Unified diff format |
| Apply changes | N/A | `CodexPatch` | `GeminiPatch` | Multi-file patches |
| Reasoning | `thinking` | `CodexReasoning` | `GeminiReasoning` | Extended thinking |
| Search files | `Glob`/`Grep` | N/A | `search` | Different search APIs |

### 7.2 Tool View Component Mapping

**Location:** `sources/components/tools/views/_all.tsx`

```typescript
export function getToolViewComponent(toolName: string) {
    switch (toolName) {
        // Codex-specific views
        case 'CodexBash':
            return CodexBashView;
        case 'CodexPatch':
            return CodexPatchView;
        case 'CodexDiff':
            return CodexDiffView;

        // Gemini-specific views
        case 'edit':
            return GeminiEditView;
        case 'execute':
            return GeminiExecuteView;

        // ... other views ...

        default:
            return null;
    }
}
```

**Total Custom Components:**
- **Codex:** 3 custom view components (Bash, Patch, Diff)
- **Gemini:** 2 custom view components (Edit, Execute)

---

## 8. Development Timeline

### 8.1 Recent Work (from CHANGELOG.md)

**Version 5 (2025-12-22):**
```markdown
- We are working on adding Gemini support using ACP and hopefully fixing
  codex stability issues using the same approach soon! Stay tuned.
- Removed model configurations from agents.
```

**Version 4 (2025-09-12):**
```markdown
- Introduced Codex support for advanced AI-powered code completion and
  generation capabilities.
- Implemented Daemon Mode as the new default.
```

**Analysis:**
- Codex added in Version 4 (Sep 2025)
- Gemini experimental in Version 5 (Dec 2025)
- ACP protocol built to support both

### 8.2 Git Commit Evidence

**From git log:**
```
45a9038 Merge pull request #376 from Scoteezy/feat/acp-gemini-integration
2bfe284 feat(gemini): improve tool display for edit and execute actions
308c3bf fix(tools): hide unknown Gemini tools automatically
ca6a056 feat: add ACP integration for Gemini with model selection and improved UX
```

**Key Commits:**
1. **#376** - Main ACP/Gemini integration PR
2. **2bfe284** - Gemini tool UI improvements
3. **308c3bf** - Auto-hide unknown tools
4. **ca6a056** - Model selection UX

---

## 9. Integration Architecture Comparison

### 9.1 Claude Code Integration

| Aspect | Implementation |
|--------|---------------|
| **Method** | Documentation-based (CLAUDE.md) |
| **Configuration** | `.claude/` directory |
| **Custom Agents** | i18n-translator agent |
| **Permissions** | `.claude/settings.json` |
| **Communication** | Direct Claude Code CLI |
| **Format** | Underscore (tool_use, tool_result) |

### 9.2 Codex Integration

| Aspect | Implementation |
|--------|---------------|
| **Method** | Protocol-based (ACP) |
| **Configuration** | Runtime flavor detection |
| **Custom Agents** | No custom agents |
| **Permissions** | Standard permission system |
| **Communication** | ACP protocol translation |
| **Format** | Hyphenated (tool-call, tool-call-result) |

### 9.3 Gemini Integration

| Aspect | Implementation |
|--------|---------------|
| **Method** | Protocol-based (ACP) |
| **Configuration** | Runtime flavor detection |
| **Custom Agents** | No custom agents |
| **Permissions** | Standard permission system |
| **Communication** | ACP protocol translation |
| **Format** | Hyphenated + nested structures |

### 9.4 Key Differences

**Claude:** Static configuration files, custom agents, documentation-driven
**Codex:** Runtime protocol adaptation, parsed command metadata
**Gemini:** Runtime protocol adaptation, nested data structures

---

## 10. Strengths of Multi-Agent Architecture

### 10.1 Protocol Abstraction

✅ **Unified Protocol** - ACP supports multiple providers with single codebase
✅ **Future-proof** - Easy to add new agent providers
✅ **Type-safe** - Zod schemas validate all messages
✅ **Robust** - Handles format variations (hyphenated vs underscore)
✅ **Backward Compatible** - Claude continues to work with direct format

### 10.2 UI Adaptation

✅ **Custom Components** - Agent-specific UI (CodexBashView, GeminiEditView)
✅ **Auto-hiding** - Unknown tools automatically hidden (Gemini)
✅ **Flavor Detection** - UI adapts based on agent type
✅ **Consistent UX** - All agents look similar to users
✅ **Specialized Rendering** - Parsed commands (Codex), nested data (Gemini)

### 10.3 Extensibility

✅ **Tool Mapping** - Easy to add new tools per agent
✅ **Message Types** - New ACP message types easily added
✅ **Model Selection** - Per-agent model configuration
✅ **No Breaking Changes** - Backward compatible with existing agents
✅ **Provider Slot** - `opencode` reserved for future agents

---

## 11. Integration Workflow

### 11.1 Session Creation with Codex

1. **User selects "Codex" profile** in NewSessionWizard
2. **Wizard shows OpenAI API key input** (if needed)
3. **Session starts with `flavor: "codex"`** in metadata
4. **ACP messages flow** from Codex CLI to Happy mobile app
5. **Protocol layer normalizes** hyphenated format to internal format
6. **UI renders** using CodexBashView, CodexPatchView, etc.

### 11.2 Session Creation with Gemini

1. **User selects "Gemini" profile** in NewSessionWizard
2. **Wizard shows Gemini model options** (2.5-pro, 2.5-flash, 2.5-flash-lite)
3. **User configures Gemini API key** (if needed)
4. **Session starts with `flavor: "gemini"`** in metadata
5. **ACP messages flow** from Gemini via ACP
6. **Protocol layer normalizes** nested structures
7. **UI renders** using GeminiEditView, GeminiExecuteView
8. **Unknown tools auto-hidden** via flavor detection

### 11.3 Message Flow

```
AI Agent (Claude/Codex/Gemini)
    ↓
Agent CLI (native format or ACP)
    ↓
Happy Server (WebSocket + encryption)
    ↓
Happy Mobile App
    ↓
Protocol Normalization (typesRaw.ts)
    │
    ├─ Claude: Direct underscore format
    ├─ Codex: ACP hyphenated → underscore
    └─ Gemini: ACP nested → underscore
    ↓
Message Storage (sync/storage.ts)
    ↓
UI Rendering
    │
    ├─ Claude: Standard tool views
    ├─ Codex: CodexBashView, CodexPatchView, CodexDiffView
    └─ Gemini: GeminiEditView, GeminiExecuteView
    ↓
User sees formatted output
```

---

## 12. Security Considerations

### 12.1 Protocol Validation

✅ **Zod Schemas** - All ACP messages validated with TypeScript types
✅ **Passthrough Fields** - Unknown fields preserved for future compatibility
✅ **Error Handling** - Validation errors logged but don't crash app
✅ **Provider Enum** - Limited to known providers (gemini, codex, claude, opencode)

### 12.2 API Key Handling

**Location:** `sources/components/NewSessionWizard.tsx:1074-1098`

```typescript
// Only add user-provided API keys if they're non-empty
// This preserves CLI environment variable precedence
const userApiKeys = profileApiKeys[selectedProfileId];
if (userApiKeys) {
    Object.entries(userApiKeys).forEach(([key, value]) => {
        // Only override if user provided a non-empty value
        if (value && value.trim().length > 0) {
            environmentVariables![key] = value;
        }
    });
}
```

**Security Features:**
- API keys stored in session (not persisted long-term)
- Empty wizard fields preserve CLI env vars
- Secure text input for API keys
- No server-side storage of keys
- Keys encrypted in transit (end-to-end encryption)

---

## 13. Testing & Debugging

### 13.1 Flavor Detection

**How to test:**
```typescript
// Check session flavor
const session = getSession(sessionId);
console.log('Agent flavor:', session.metadata?.flavor);
// Output: "codex" | "gemini" | null (claude)
```

### 13.2 Protocol Validation

**Location:** `sources/sync/typesRaw.ts:404-410`

```typescript
let parsed = rawRecordSchema.safeParse(raw);
if (!parsed.success) {
    console.error('=== VALIDATION ERROR ===');
    console.error('Zod issues:', JSON.stringify(parsed.error.issues, null, 2));
    console.error('Raw message:', JSON.stringify(raw, null, 2));
    console.error('=== END ERROR ===');
    return null;
}
```

**Benefit:** Detailed error logging for protocol debugging

---

## 14. Limitations & Future Work

### 14.1 Current Limitations

⚠️ **No Codex/Gemini documentation** - No `.codex/` or `.gemini/` config directories
⚠️ **No custom agents** - Unlike Claude's i18n-translator
⚠️ **Model configuration removed** - App relies on CLI defaults
⚠️ **Gemini experimental** - Version 5 mentions "working on adding Gemini support"
⚠️ **No tests found** - No unit tests for ACP protocol or agent-specific components

### 14.2 Future Enhancements

**Potential Additions:**
1. Codex-specific slash commands
2. Gemini-specific slash commands
3. Better tool descriptions in UI
4. Model presets per agent
5. Agent-specific error handling
6. Extended thinking/reasoning UI
7. Tests for ACP protocol
8. Integration tests for multi-agent sessions
9. `opencode` provider integration (reserved slot)

---

## 15. Integration Quality Metrics

### 15.1 Code Coverage

| Component | Codex Support | Gemini Support |
|-----------|--------------|----------------|
| Protocol Layer | ⭐⭐⭐⭐⭐ (Complete) | ⭐⭐⭐⭐⭐ (Complete) |
| UI Components | ⭐⭐⭐⭐⭐ (3 views) | ⭐⭐⭐⭐ (2 views) |
| Tool Mapping | ⭐⭐⭐⭐⭐ (4+ tools) | ⭐⭐⭐⭐⭐ (10+ tools) |
| Model Selection | ⚠️ (CLI only) | ⭐⭐⭐⭐ (3 models) |
| Documentation | ⭐⭐ (Changelog) | ⭐⭐ (Changelog) |
| Testing | ⚠️ (No tests) | ⚠️ (No tests) |

### 15.2 Integration Completeness

**Implemented:**
✅ ACP protocol normalization
✅ Hyphenated format support
✅ Tool-call handling
✅ File edit operations
✅ Terminal command execution
✅ Thinking/reasoning display
✅ Flavor detection
✅ Agent-specific UI components
✅ Auto-hidden unknown tools (Gemini)
✅ Parsed command metadata (Codex)
✅ Nested data extraction (Gemini)

**Not Implemented:**
❌ Agent-specific documentation (`.codex/`, `.gemini/`)
❌ Custom agents for Codex/Gemini
❌ Agent-specific permissions
❌ Agent-specific slash commands
❌ Unit tests
❌ Integration tests

---

## 16. Recommendations

### 16.1 For Happy Developers

1. **Add Agent Documentation**
   - Create `.codex/README.md` explaining Codex-specific features
   - Create `.gemini/README.md` for Gemini features
   - Document ACP protocol for contributors
   - Add CLI setup instructions per agent

2. **Improve Testing**
   - Add unit tests for ACP protocol normalization
   - Add tests for Codex-specific components
   - Add tests for Gemini-specific components
   - Test flavor detection logic
   - Test format translation (hyphenated ↔ underscore)

3. **Enhanced Tool UI**
   - Add tool descriptions in UI
   - Improve diff rendering
   - Show parsed command metadata more prominently (Codex)
   - Better nested structure display (Gemini)

4. **Model Configuration**
   - Consider restoring in-app model selection
   - Add model descriptions and capabilities
   - Show model limits/quotas

### 16.2 For Other Projects

**If building multi-agent support:**

✅ **Use protocol abstraction** - Don't hardcode agent-specific logic
✅ **Runtime flavor detection** - Adapt UI based on agent type
✅ **Normalize message formats** - Single internal representation
✅ **Custom UI components** - Agent-specific rendering when needed
✅ **Type safety** - Use Zod/TypeScript for validation
✅ **Format translation** - Handle different naming conventions gracefully
✅ **Extensibility** - Reserve provider slots for future agents

---

## 17. Conclusion

### 17.1 Overall Assessment

The Happy repository demonstrates **production-ready multi-agent support** through a sophisticated **Agent Communication Protocol (ACP)** architecture combined with documentation-based integration for Claude Code.

**Integration Quality:** ⭐⭐⭐⭐⭐ (5/5)

**Key Achievements:**
1. Unified protocol supporting 3+ AI agents (Claude, Codex, Gemini)
2. Type-safe message normalization
3. Agent-specific UI components (5 custom views)
4. Auto-adapting behavior based on agent flavor
5. Seamless format translation (hyphenated ↔ underscore)
6. Hybrid architecture (documentation + protocol)
7. Extensible design with reserved provider slots

### 17.2 Integration Summary by Agent

**Claude:** Documentation-heavy, custom agents, static configuration, mature
**Codex:** Protocol-driven, parsed metadata, production-ready
**Gemini:** Protocol-driven, nested structures, experimental

### 17.3 Innovation Highlights

**Novel Approaches:**
- **ACP Protocol** - Unified format for multiple agent providers
- **Flavor Detection** - Runtime UI adaptation
- **Auto-Hide Unknown Tools** - Prevents UI clutter (Gemini)
- **Parsed Command Metadata** - Enhanced bash command display (Codex)
- **Nested Data Extraction** - Handles complex structures (Gemini)
- **Hybrid Integration** - Combines documentation (Claude) + protocol (Codex/Gemini)
- **Format Translation** - Hyphenated ↔ Underscore normalization

### 17.4 Production Readiness

| Agent | Status | Evidence |
|-------|--------|----------|
| Claude | Production | Comprehensive CLAUDE.md, custom agents, mature |
| Codex | Production | Version 4 release, 3 custom UI components |
| Gemini | Experimental | Version 5 "working on adding", 2 custom UI components |

**Overall:** Production-ready with minor experimental features

### 17.5 Learning Points for Other Projects

**What to Emulate:**

1. **Protocol Abstraction** - Build unified protocol instead of agent-specific code
2. **Type Safety** - Use Zod schemas for runtime validation
3. **Runtime Adaptation** - Detect agent type and adapt UI dynamically
4. **Custom Components** - Create agent-specific UI where needed
5. **Graceful Degradation** - Auto-hide unknown features
6. **Hybrid Approach** - Combine documentation and protocol methods
7. **Format Normalization** - Handle different naming conventions transparently
8. **Extensibility** - Design for future agents from the start

**This Project as a Template:**

The Happy repository serves as an **excellent reference** for building **multi-agent mobile clients**. The ACP protocol architecture is particularly innovative and demonstrates how to support multiple AI coding agents with different formats and capabilities in a single codebase.

The combination of:
- Documentation-based integration (Claude)
- Protocol-based integration (Codex, Gemini)
- Agent-specific UI components
- Runtime flavor detection
- Type-safe message normalization

...creates a robust, extensible foundation for AI code agent clients.

---

## Appendix A: File Locations

### Core Protocol Files
```
sources/sync/
├── typesRaw.ts                 # ACP protocol definition (850+ lines)
├── storageTypes.ts             # Session metadata with flavor field
└── sync.ts                     # Sync engine using ACP

sources/components/tools/
├── knownTools.tsx              # Tool definitions (950+ lines)
│                               # - Claude tools
│                               # - Codex tools (CodexBash, CodexPatch, etc.)
│                               # - Gemini tools (GeminiEdit, GeminiExecute, etc.)
├── ToolView.tsx                # Tool rendering with flavor detection
└── views/
    ├── CodexBashView.tsx       # Codex bash/shell tool UI
    ├── CodexPatchView.tsx      # Codex file patching UI
    ├── CodexDiffView.tsx       # Codex diff display UI
    ├── GeminiEditView.tsx      # Gemini edit tool UI
    ├── GeminiExecuteView.tsx   # Gemini execute tool UI
    └── _all.tsx                # Tool view component mapping

sources/components/
└── NewSessionWizard.tsx        # Session creation with multi-agent support
```

### Configuration Files
```
.claude/
├── agents/
│   └── i18n-translator.md      # Custom Claude agent
└── settings.json               # Claude permissions

CLAUDE.md                       # Claude Code integration docs
CHANGELOG.md                    # Version history (Codex V4, Gemini V5)
```

### Git History
```
45a9038 Merge pull request #376 from Scoteezy/feat/acp-gemini-integration
2bfe284 feat(gemini): improve tool display for edit and execute actions
308c3bf fix(tools): hide unknown Gemini tools automatically
ca6a056 feat: add ACP integration for Gemini with model selection and improved UX
```

---

## Appendix B: ACP Message Types

### Supported ACP Message Types

| Type | Purpose | Normalized To | Primary User |
|------|---------|---------------|--------------|
| `message` | Text messages | `text` | Codex, Gemini |
| `reasoning` | Reasoning steps | `text` | Codex, Gemini |
| `thinking` | Extended thinking | `thinking` | Codex, Gemini |
| `tool-call` | Tool invocation | `tool-call` | Codex, Gemini |
| `tool-result` | Tool result | `tool-result` | Codex, Gemini |
| `tool-call-result` | Legacy format | `tool-result` | Codex, Gemini |
| `file-edit` | File editing | `tool-call` | Codex, Gemini |
| `terminal-output` | Command output | `tool-result` | Codex |
| `task_started` | Task lifecycle | (skipped) | Codex |
| `task_complete` | Task lifecycle | (skipped) | Codex |
| `turn_aborted` | Task lifecycle | (skipped) | Codex |
| `permission-request` | Permission dialog | `tool-call` | Codex, Gemini |
| `token_count` | Usage metrics | (skipped) | Codex |

---

## Appendix C: Metrics

**Protocol Implementation:**
- **ACP Schema:** 60+ lines (typesRaw.ts:210-271)
- **Normalization Logic:** 170+ lines (typesRaw.ts:642-815)
- **Tool Definitions:**
  - Codex: 130+ lines (knownTools.tsx:440-518, 687-817)
  - Gemini: 280+ lines (knownTools.tsx:543-884)
- **Custom UI Components:** 5 files
  - Codex: 3 (CodexBashView, CodexPatchView, CodexDiffView)
  - Gemini: 2 (GeminiEditView, GeminiExecuteView)

**Agent Support:**
- **Total Agents:** 3 (Claude, Codex, Gemini)
- **ACP Providers:** 4 slots (codex, gemini, claude, opencode)
- **Custom Claude Agents:** 1 (i18n-translator)

**Model Support:**
- **Gemini Models:** 3 (pro, flash, flash-lite)
- **Codex Models:** CLI configured
- **Claude Models:** Claude Code configured

**Git Activity:**
- **Codex:** Version 4 (Sep 2025)
- **Gemini:** Version 5 (Dec 2025), PR #376
- **Recent Commits:** 4+ Gemini-related, Codex mature

---

**Generated:** January 15, 2026
**Repository:** https://github.com/slopus/happy
**Analysis Depth:** Comprehensive (All Agents + Protocol + UI)
**Focus:** Multi-agent architecture (Claude, Codex, Gemini)
