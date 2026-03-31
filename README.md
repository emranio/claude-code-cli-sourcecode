# Claude Code CLI

**The source of Claude Cli was leaked by sopurcemap and found on Chaofan's X account.** 

https://x.com/Fried\_rice/status/2038894956459290963

# Claude Code CLI — Architecture & Structural Guide

## LLM generated details about this repo aka Clause Code Cli.

## Overview

Claude Code CLI is a terminal-based AI coding assistant built with **TypeScript**, **React (Ink)**, and **Bun**. It provides an interactive REPL, an SDK/headless mode, and a remote bridge system for connecting to cloud-managed sessions. The codebase lives entirely under `src/` and uses feature flags (`bun:bundle` `feature()`) for dead-code elimination of internal/experimental modules.

* * *

## Table of Contents

1.  [Entry Points & Boot Sequence](#1-entry-points--boot-sequence)
2.  [Core Engine](#2-core-engine)
3.  [State Management](#3-state-management)
4.  [Tools System](#4-tools-system)
5.  [Commands (Slash Commands)](#5-commands-slash-commands)
6.  [Tasks System](#6-tasks-system)
7.  [Services Layer](#7-services-layer)
8.  [Bridge / Remote Sessions](#8-bridge--remote-sessions)
9.  [UI Layer (Ink + Components)](#9-ui-layer-ink--components)
10.  [Plugins & Skills](#10-plugins--skills)
11.  [Utilities & Cross-Cutting Concerns](#11-utilities--cross-cutting-concerns)
12.  [Supporting Subsystems](#12-supporting-subsystems)
13.  [Directory Map](#13-directory-map)

* * *

## 1\. Entry Points & Boot Sequence

### `src/entrypoints/cli.tsx` — Bootstrap Entrypoint

The actual process entry point. Handles fast-path exits (`--version`, `--dump-system-prompt`, `--chrome-native-host`) with zero imports, then dynamically loads the full CLI via `src/main.tsx`. Also sets up environment tweaks (COREPACK, heap size for remote, ablation baselines).

### `src/main.tsx` — Full CLI Initialization

The heavyweight entry. Responsibilities:

-   **Side-effect imports at top**: Startup profiling, MDM raw reads, keychain prefetches — all fire in parallel before module evaluation completes.
-   **Commander.js CLI parsing**: Defines all CLI flags (`--model`, `--permission-mode`, `--resume`, `--print`, `--sdk`, etc.) via `@commander-js/extra-typings`.
-   **Authentication & configuration**: OAuth, API keys, GrowthBook feature flags, policy limits, remote managed settings.
-   **Session setup**: Session ID generation, conversation resume, worktree creation, MCP server initialization.
-   **Mode dispatch**: Routes to one of:
    -   **REPL mode** → `launchRepl()` (interactive terminal UI)
    -   **Print/SDK mode** → `QueryEngine` (headless, structured output)
    -   **Bridge mode** → `bridgeMain()` (cloud-managed sessions)
    -   **MCP server mode** → `src/entrypoints/mcp.ts`
    -   **Assistant (Kairos) mode** → Feature-gated assistant module

### `src/entrypoints/init.ts` — One-Time Initialization

Memoized `init()` function that runs exactly once:

-   OpenTelemetry setup (metrics, tracing, logs)
-   Proxy configuration (`configureGlobalAgents`)
-   mTLS configuration
-   OAuth account info population
-   Policy limits and remote managed settings loading
-   API preconnection
-   Repository detection

### `src/setup.ts` — Session Setup

Called after init, per-session:

-   Working directory resolution and git root detection
-   Session ID assignment and session memory initialization
-   Worktree creation (if `--worktree` flag)
-   UDS messaging server startup
-   Hook configuration snapshots
-   File watcher initialization

* * *

## 2\. Core Engine

### `src/query.ts` — Query Loop (REPL Path)

The main agentic loop for interactive sessions. Orchestrates:

-   User input processing and slash command detection
-   Streaming API calls to Claude via `src/services/api/claude.ts`
-   Tool use execution and result collection
-   Auto-compaction when context window fills
-   Message queue management for batched inputs
-   Interruption handling (Ctrl+C)

### `src/QueryEngine.ts` — Query Engine (SDK/Headless Path)

A class-based extraction of the query lifecycle for headless/SDK usage:

-   `QueryEngineConfig` — configuration for a conversation session
-   `QueryEngine` class — owns messages, file cache, usage tracking, abort controller
-   `submitMessage()` — starts a new turn within the conversation
-   Handles permission denials, snip boundaries, and session persistence
-   Used by print mode, SDK mode, and agent subprocesses

### `src/Tool.ts` — Tool Type Definitions

Central type definitions for the tool system:

-   `Tool` interface: name, description, input schema, validation, execution
-   `ToolUseContext`: the runtime context passed to every tool execution (model, commands, tools, MCP clients, abort controller, app state, etc.)
-   `ToolPermissionContext`: permission mode, always-allow/deny/ask rules, working directories
-   `SetToolJSXFn`: callback for tools to render inline UI
-   Helper types: `ValidationResult`, `QueryChainTracking`, `CompactProgressEvent`
-   `findToolByName()`, `toolMatchesName()` — tool lookup utilities

* * *

## 3\. State Management

### `src/state/store.ts` — Generic Store

A minimal Zustand-like store implementation:

-   `createStore<T>(initialState, onChange?)` → `{ getState, setState, subscribe }`
-   Immutable state updates via updater functions
-   Listener notification on state change
-   `onChange` callback for side-effect reactions

### `src/state/AppStateStore.ts` — Application State

The central `AppState` type (wrapped in `DeepImmutable`):

-   **Session**: messages, verbose mode, model settings, status line
-   **UI**: expanded view, selected agent index, footer items, prompt input
-   **Tools**: tool permission context, MCP clients/resources, agent definitions
-   **Features**: speculation state, thinking config, effort, fast mode
-   **Background**: tasks, teammates, session hooks state
-   **Plugins/Skills**: loaded plugins, plugin errors
-   `getDefaultAppState()` — factory for initial state

### `src/state/AppState.tsx` — React Context Provider

React context wrapper that bridges the store to the Ink component tree. Components use hooks to subscribe to state slices.

### `src/bootstrap/state.ts` — Global Bootstrap State

Module-level mutable state for values needed before the store exists:

-   `cwd`, `originalCwd`, `projectRoot`
-   `sessionId`, `totalCostUSD`, `modelUsage`
-   `isInteractive`, `clientType`, `sdkBetas`
-   Counters and meters (OpenTelemetry)
-   Model override, settings paths, channel entries
-   Accessed via exported getter/setter functions (e.g., `getSessionId()`, `setCwd()`)

### `src/state/onChangeAppState.ts` — State Change Reactions

Side-effect handler triggered by `store.onChange`, dispatches reactions to state transitions (e.g., updating session storage, triggering analytics).

* * *

## 4\. Tools System

### `src/tools.ts` — Tool Registry

`getTools()` returns the complete array of available tools. Each tool is a separate module under `src/tools/<ToolName>/`:

| Tool | Purpose |
| --- | --- |
| BashTool | Execute shell commands |
| FileReadTool | Read file contents |
| FileWriteTool | Write/create files |
| FileEditTool | Search-and-replace edits |
| GlobTool | File pattern matching |
| GrepTool | Text/regex search |
| WebFetchTool | Fetch web page content |
| WebSearchTool | Web search |
| AgentTool | Spawn sub-agent (agentic delegation) |
| SkillTool | Invoke registered skills |
| TaskOutputTool | Read background task output |
| TaskStopTool | Stop background tasks |
| NotebookEditTool | Edit Jupyter notebooks |
| BriefTool | Generate brief summaries |
| LSPTool | Language Server Protocol queries |
| MCPTool | Invoke MCP server tools (dynamic) |
| TodoWriteTool | Manage todo lists |
| EnterPlanModeTool | Enter planning mode |
| ExitPlanModeV2Tool | Exit planning mode |
| EnterWorktreeTool | Switch to git worktree |
| ExitWorktreeTool | Exit worktree |
| AskUserQuestionTool | Prompt user for input |
| ToolSearchTool | Search available tools |
| ListMcpResourcesTool | List MCP resources |
| ReadMcpResourceTool | Read MCP resources |
| TeamCreateTool | Create teammate (swarm) |
| TeamDeleteTool | Delete teammate |
| SendMessageTool | Send message to teammate |
| TungstenTool | Internal tooling |

**Feature-gated tools** (conditionally loaded via `feature()` or `process.env.USER_TYPE`):

-   `REPLTool`, `SuggestBackgroundPRTool` (ant-only)
-   `SleepTool`, `SendUserFileTool`, `PushNotificationTool` (Kairos)
-   `CronCreateTool/CronDeleteTool/CronListTool` (agent triggers)
-   `MonitorTool`, `RemoteTriggerTool`, `SubscribePRTool`

### `src/tools/shared/` — Shared Tool Utilities

Common helpers used across multiple tools.

* * *

## 5\. Commands (Slash Commands)

### `src/commands.ts` — Command Registry

`getCommands()` returns all registered slash commands. Each command is a module under `src/commands/<name>/`:

**Core commands**: `/help`, `/clear`, `/compact`, `/config`, `/cost`, `/diff`, `/doctor`, `/init`, `/resume`, `/status`, `/vim`, `/theme`, `/model`, `/permissions`

**Git/PR commands**: `/commit`, `/commit-push-pr`, `/review`, `/security-review`, `/pr_comments`

**Session commands**: `/session`, `/share`, `/rename`, `/export`, `/teleport`

**Agent/Team commands**: `/agents`, `/tasks`, `/bridge`

**Context commands**: `/context`, `/add-dir`, `/memory`, `/skills`

**System commands**: `/login`, `/logout`, `/install`, `/doctor`, `/version`, `/usage`, `/env`

**Feature-gated**: `/proactive`, `/brief`, `/assistant`, `/bridge`, `/voice`, `/bughunter`, `/ultraplan`, `/ultrareview`

* * *

## 6\. Tasks System

### `src/Task.ts` — Task Types & Interfaces

Defines the task abstraction for background work:

-   `TaskType`: `local_bash` | `local_agent` | `remote_agent` | `in_process_teammate` | `local_workflow` | `monitor_mcp` | `dream`
-   `TaskStatus`: `pending` | `running` | `completed` | `failed` | `killed`
-   `TaskStateBase`: metadata (id, type, status, description, timestamps, output file)
-   `Task` interface: `name`, `type`, `kill()`
-   Task ID generation with type-prefixed random IDs

### `src/tasks.ts` — Task Registry

`getAllTasks()` / `getTaskByType()` — returns task implementations:

-   `LocalShellTask` — background shell command execution
-   `LocalAgentTask` — local sub-agent process
-   `RemoteAgentTask` — remote cloud agent
-   `DreamTask` — dream/speculation task
-   `LocalWorkflowTask` — workflow scripts (feature-gated)
-   `MonitorMcpTask` — MCP server monitoring (feature-gated)

Task implementations live under `src/tasks/<TaskName>/`.

* * *

## 7\. Services Layer

### `src/services/api/` — API Communication

-   **`claude.ts`**: Core API wrapper — streaming message creation, tool schema conversion, model selection, betas, retry logic. Interfaces with the Anthropic SDK.
-   **`client.ts`**: Anthropic SDK client construction for multiple providers (Direct API, AWS Bedrock, Azure Foundry, Vertex AI). Handles auth, proxy, credentials refresh.
-   **`errors.ts`**: API error classification, retry categorization, prompt-too-long detection.
-   **`withRetry.ts`**: Retry logic with exponential backoff, fallback model support.
-   **`bootstrap.ts`**: Bootstrap data fetching.
-   **`filesApi.ts`**: Session file download/upload.
-   **`logging.ts`**: API request/response logging, usage tracking.

### `src/services/mcp/` — Model Context Protocol

-   **`client.ts`**: MCP client management — connects to configured servers, fetches tools/commands/resources.
-   **`config.ts`**: MCP server configuration parsing from multiple sources (project, user, enterprise).
-   **`types.ts`**: MCP connection types, server configs, resources.
-   **`auth.ts`**: MCP OAuth authentication flows.
-   **`channelPermissions.ts`**: Permission management for MCP channels.
-   **`elicitationHandler.ts`**: URL elicitation handling for MCP tool errors.
-   **`MCPConnectionManager.tsx`**: React component for managing MCP connections.

### `src/services/analytics/` — Analytics & Telemetry

-   **`index.ts`**: `logEvent()` — central analytics dispatch.
-   **`growthbook.ts`**: GrowthBook feature flag integration.
-   **`datadog.ts`**: Datadog metrics exporter.
-   **`sink.ts`**: Analytics sink management.
-   **`firstPartyEventLogger.ts`**: First-party event logging via OpenTelemetry.

### `src/services/compact/` — Context Compaction

Auto-compaction and reactive compaction to manage context window limits.

### `src/services/lsp/` — Language Server Protocol

LSP server management for code intelligence features.

### `src/services/plugins/` — Plugin Management

Plugin CLI commands and lifecycle management.

### `src/services/policyLimits/` — Policy Limits

Enterprise policy limits enforcement.

### `src/services/remoteManagedSettings/` — Remote Settings

Remote managed settings for enterprise deployments.

### `src/services/oauth/` — OAuth

OAuth client for Claude.ai authentication.

### `src/services/tips/` — Tips

Contextual tips and suggestions.

### `src/services/PromptSuggestion/` — Prompt Suggestions

Prompt suggestion generation.

### `src/services/SessionMemory/` — Session Memory

Per-session memory management.

* * *

## 8\. Bridge / Remote Sessions

### `src/bridge/` — Remote Bridge System

Enables cloud-managed Claude Code sessions:

-   **`bridgeMain.ts`**: Main bridge loop — polls for work, spawns sessions, manages lifecycle. Implements backoff, capacity wake, JWT refresh.
-   **`bridgeApi.ts`**: HTTP client for bridge API (register worker, poll, report status).
-   **`bridgeConfig.ts`** / **`envLessBridgeConfig.ts`**: Bridge configuration.
-   **`bridgeMessaging.ts`**: Message passing between bridge and sessions.
-   **`bridgePermissionCallbacks.ts`**: Permission handling in bridge context.
-   **`bridgeUI.ts`**: Terminal UI for bridge status.
-   **`sessionRunner.ts`**: Spawns and manages individual session processes.
-   **`replBridge.ts`** / **`replBridgeHandle.ts`** / **`replBridgeTransport.ts`**: Bridge transport for REPL sessions.
-   **`jwtUtils.ts`**: JWT token management and refresh scheduling.
-   **`trustedDevice.ts`**: Trusted device token management.
-   **`workSecret.ts`**: Work secret handling for bridge authentication.
-   **`capacityWake.ts`**: Capacity-based wake system.
-   **`types.ts`**: Bridge type definitions.

### `src/remote/` — Remote Session Management

-   **`RemoteSessionManager.ts`**: Manages remote sessions.
-   **`SessionsWebSocket.ts`**: WebSocket connection for real-time session communication.

### `src/server/` — Direct Connect Server

-   **`createDirectConnectSession.ts`**: Creates direct-connect sessions.
-   **`directConnectManager.ts`**: Manages direct connections.

* * *

## 9\. UI Layer (Ink + Components)

### `src/ink.ts` — Ink Wrapper

Re-exports the custom Ink rendering system with automatic `ThemeProvider` wrapping. Exports `render()`, `createRoot()`, and all design system primitives.

### `src/ink/` — Custom Ink Fork

A forked/customized version of Ink (React for CLIs):

-   **`root.ts`** / **`ink.tsx`**: Root rendering and reconciler setup.
-   **`reconciler.ts`**: Custom React reconciler for terminal output.
-   **`dom.ts`**: Virtual DOM for terminal nodes.
-   **`renderer.ts`** / **`frame.ts`**: Frame rendering pipeline.
-   **`output.ts`** / **`render-to-screen.ts`**: Screen output management.
-   **`components/`**: Base components (Box, Text, Button, Link, Newline, Spacer, etc.).
-   **`hooks/`**: Terminal hooks (useInput, useApp, useStdin, useTerminalViewport, useSelection, etc.).
-   **`events/`**: Event system (click, input, terminal focus).
-   **`layout/`**: Layout engine (Yoga-based flexbox).
-   **`termio/`**: Terminal I/O primitives (DEC sequences, OSC, cursor control).

### `src/screens/` — Top-Level Screens

-   **`REPL.tsx`**: The main interactive REPL screen — the primary user interface.
-   **`Doctor.tsx`**: Diagnostic/doctor screen.
-   **`ResumeConversation.tsx`**: Session resume UI.

### `src/components/` — React Components (~180+ components)

Major component categories:

**Core UI**: `App.tsx`, `Messages.tsx`, `MessageRow.tsx`, `MessageResponse.tsx`, `PromptInput/`, `StatusLine.tsx`, `Spinner/`, `Stats.tsx`

**Dialogs**: `TrustDialog/`, `AutoModeOptInDialog.tsx`, `BypassPermissionsModeDialog.tsx`, `CostThresholdDialog.tsx`, `ExitFlow.tsx`, `MCPServerApprovalDialog.tsx`, `InvalidSettingsDialog.tsx`, `HistorySearchDialog.tsx`, `GlobalSearchDialog.tsx`

**Code display**: `HighlightedCode/`, `StructuredDiff/`, `Markdown.tsx`, `FileEditToolDiff.tsx`

**Tool UI**: `ToolUseLoader.tsx`, `BashModeProgress.tsx`, `AgentProgressLine.tsx`, `CoordinatorAgentStatus.tsx`

**Settings/Config**: `Settings/`, `ModelPicker.tsx`, `ThemePicker.tsx`, `OutputStylePicker.tsx`, `LanguagePicker.tsx`

**Special features**: `VirtualMessageList.tsx`, `TextInput.tsx`, `VimTextInput.tsx`, `CompactSummary.tsx`, `TokenWarning.tsx`

**Design system**: `design-system/` (ThemeProvider, ThemedBox, ThemedText, color system)

### `src/components/hooks/` — Component-Level Hooks

Hooks specific to component rendering (distinct from the top-level `src/hooks/`).

* * *

## 10\. Plugins & Skills

### `src/plugins/`

-   **`builtinPlugins.ts`**: Defines built-in plugins.
-   **`bundled/`**: Bundled plugin implementations.
-   Plugin loading and lifecycle managed via `src/utils/plugins/`.

### `src/skills/`

-   **`bundledSkills.ts`** / **`bundled/`**: Built-in skill definitions.
-   **`loadSkillsDir.ts`**: Loads skills from `.claude/skills/` directories.
-   **`mcpSkillBuilders.ts`**: Creates skill wrappers for MCP tools.

### `src/outputStyles/`

-   **`loadOutputStylesDir.ts`**: Custom output style loading.

* * *

## 11\. Utilities & Cross-Cutting Concerns

### `src/utils/` — Utility Library (~300+ files)

The largest directory, organized by domain:

**Authentication**: `auth.ts`, `authPortable.ts`, `authFileDescriptor.ts`, `secureStorage/`

**Configuration**: `config.ts`, `configConstants.ts`, `settings/`, `managedEnv.ts`

**File Operations**: `fsOperations.ts`, `fileStateCache.ts`, `fileHistory.ts`, `fileRead.ts`, `file.ts`

**Git Integration**: `git.ts`, `git/`, `gitDiff.ts`, `gitSettings.ts`, `github/`, `worktree.ts`

**Model Management**: `model/` (model selection, providers, deprecation, capabilities, strings)

**Permissions**: `permissions/` (PermissionMode, filesystem, denial tracking, auto-mode state, permission setup)

**Messages**: `messages.ts`, `messages/` (creation, normalization, mappers, system init)

**Hooks System**: `hooks/`, `hooks.ts` (session hooks, file changed watcher, post-sampling hooks)

**Process Management**: `process.ts`, `Shell.ts`, `ShellCommand.ts`, `bash/`, `execFileNoThrow.ts`

**Session Management**: `sessionStorage.ts`, `sessionRestore.ts`, `sessionState.ts`, `sessionStart.ts`

**Context Building**: `queryContext.ts`, `context.ts`, `systemPrompt.ts`, `api.ts`, `attachments.ts`

**Analytics**: `telemetry/`, `telemetryAttributes.ts`, `sinks.ts`

**Error Handling**: `errors.ts`, `log.ts`, `debug.ts`, `diagLogs.ts`

**Performance**: `startupProfiler.ts`, `headlessProfiler.ts`, `fpsTracker.ts`

**Extensions**: `plugins/`, `skills/`, `mcp/`

**UI Helpers**: `renderOptions.ts`, `theme.ts`, `format.ts`, `markdown.ts`, `diff.ts`

**Miscellaneous**: `sleep.ts`, `uuid.ts`, `json.ts`, `xml.ts`, `yaml.ts`, `array.ts`, `stringUtils.ts`, `semver.ts`

### `src/types/` — Shared Type Definitions

-   **`message.ts`**: `Message`, `UserMessage`, `AssistantMessage`, `SystemMessage`, `StreamEvent`, etc.
-   **`permissions.ts`**: `PermissionMode`, `PermissionResult`, `ToolPermissionRulesBySource`
-   **`hooks.ts`**: `HookEvent`, `HookProgress`, `PromptRequest/Response`
-   **`ids.ts`**: `SessionId`, `AgentId` branded types
-   **`plugin.ts`**: Plugin type definitions
-   **`tools.ts`**: Tool progress types (`BashProgress`, `AgentToolProgress`, etc.)
-   **`logs.ts`**: Log option types

### `src/constants/` — Constants

Product constants, OAuth config, XML tags, system prompts, query sources.

* * *

## 12\. Supporting Subsystems

### `src/coordinator/` — Coordinator Mode

Multi-agent coordinator for orchestrating parallel agent work:

-   **`coordinatorMode.ts`**: Coordinator mode logic and user context.

### `src/buddy/` — Companion System

Animated companion sprite system:

-   **`companion.ts`**: Companion logic
-   **`CompanionSprite.tsx`**: Sprite rendering
-   **`sprites.ts`**: Sprite definitions
-   **`prompt.ts`**: Companion prompts

### `src/memdir/` — Memory Directory

Persistent memory system (`.claude/memory/`):

-   **`memdir.ts`**: Memory prompt loading
-   **`findRelevantMemories.ts`**: Relevance-based memory retrieval
-   **`paths.ts`** / **`teamMemPaths.ts`**: Memory file paths
-   **`memoryScan.ts`** / **`memoryTypes.ts`**: Memory scanning and classification

### `src/keybindings/` — Keybinding System

Customizable keyboard shortcuts:

-   **`KeybindingContext.tsx`**: React context for keybindings
-   **`defaultBindings.ts`**: Default key mappings
-   **`parser.ts`** / **`match.ts`** / **`resolver.ts`**: Keybinding parsing and resolution
-   **`loadUserBindings.ts`**: User customization loading

### `src/vim/` — Vim Mode

Vim-style input handling:

-   **`motions.ts`**: Cursor motions (w, b, e, etc.)
-   **`operators.ts`**: Operators (d, c, y, etc.)
-   **`textObjects.ts`**: Text objects (iw, aw, etc.)
-   **`transitions.ts`**: Mode transitions
-   **`types.ts`**: Vim state types

### `src/voice/` — Voice Input

Voice mode support (feature-gated):

-   **`voiceModeEnabled.ts`**: Voice mode availability check.
-   Voice services in `src/services/voice.ts`, `voiceStreamSTT.ts`, `voiceKeyterms.ts`.

### `src/migrations/` — Data Migrations

Schema/config migrations for version upgrades:

-   Model migrations (Fennec→Opus, Opus→Opus1m, Sonnet1m→Sonnet45, etc.)
-   Settings migrations (auto-updates, bypass permissions, MCP servers)

### `src/cli/` — CLI Output Layer

Handles structured output for non-interactive modes:

-   **`structuredIO.ts`**: NDJSON structured output for SDK mode
-   **`print.ts`**: Print mode output formatting
-   **`remoteIO.ts`**: Remote session I/O
-   **`handlers/`** / **`transports/`**: Pluggable output handlers and transports
-   **`exit.ts`**: Exit code handling

### `src/hooks/` — React Hooks (~90+ hooks)

Application-level React hooks for the REPL UI:

-   **Session**: `useRemoteSession`, `useSessionBackgrounding`, `useTeleportResume`
-   **Input**: `useTextInput`, `useVimInput`, `usePasteHandler`, `useInputBuffer`
-   **Tools/Config**: `useCanUseTool`, `useSettings`, `useSettingsChange`, `useMergedTools`
-   **MCP**: `useMergedClients`, `useManagePlugins`, `useReplBridge`
-   **UI**: `useTerminalSize`, `useVirtualScroll`, `useBlink`, `useElapsedTime`
-   **Features**: `useVoice`, `usePromptSuggestion`, `useTypeahead`, `useSwarmInitialization`

### `src/context/` — React Contexts

-   **`QueuedMessageContext.tsx`**: Message queue context
-   **`modalContext.tsx`**: Modal dialog context
-   **`overlayContext.tsx`**: Overlay rendering context
-   **`notifications.tsx`**: Notification context
-   **`stats.tsx`**: Stats context
-   **`voice.tsx`**: Voice context
-   **`fpsMetrics.tsx`**: FPS metrics context

### `src/assistant/` — Assistant (Kairos) Mode

Feature-gated assistant mode:

-   **`sessionHistory.ts`**: Assistant session history management

### `src/query/` — Query Configuration

-   **`config.ts`**: Query configuration
-   **`deps.ts`**: Query dependencies
-   **`stopHooks.ts`**: Query stop hooks
-   **`tokenBudget.ts`**: Token budget management

### `src/native-ts/` — Native TypeScript Modules

Native/compiled TypeScript modules for performance-critical operations.

### `src/moreright/` — Additional Rightward Extensions

Supporting module extensions.

### `src/upstreamproxy/` — Upstream Proxy

Proxy configuration for outbound HTTP requests.

* * *

## 13\. Directory Map

```
src/
├── entrypoints/          # Process entry points (CLI, SDK, MCP)
│   ├── cli.tsx           #   Bootstrap → fast paths → loads main.tsx
│   ├── init.ts           #   One-time initialization (telemetry, proxy, etc.)
│   ├── mcp.ts            #   MCP server mode entry
│   ├── sdk/              #   SDK-specific entry points
│   └── agentSdkTypes.ts  #   SDK type definitions
│
├── main.tsx              # Full CLI initialization & mode dispatch
├── setup.ts              # Per-session setup (cwd, git, worktree, hooks)
├── replLauncher.tsx      # Launches REPL screen (App + REPL components)
├── dialogLaunchers.tsx   # Dialog screen launchers (resume, snapshot, etc.)
├── interactiveHelpers.tsx # Rendering helpers, setup screens, exit utilities
│
├── QueryEngine.ts        # Headless query engine (SDK/print mode)
├── query.ts              # Interactive query loop (REPL mode)
├── query/                # Query config, deps, hooks, token budget
│
├── Tool.ts               # Tool interface & types (ToolUseContext, permissions)
├── tools.ts              # Tool registry (getTools)
├── tools/                # Tool implementations (40+ tools)
│   ├── BashTool/
│   ├── FileReadTool/
│   ├── FileEditTool/
│   ├── FileWriteTool/
│   ├── AgentTool/
│   ├── SkillTool/
│   ├── GrepTool/
│   ├── GlobTool/
│   ├── WebFetchTool/
│   ├── WebSearchTool/
│   ├── MCPTool/
│   └── ...
│
├── Task.ts               # Task interface & types
├── tasks.ts              # Task registry (getAllTasks)
├── tasks/                # Task implementations
│   ├── LocalShellTask/
│   ├── LocalAgentTask/
│   ├── RemoteAgentTask/
│   └── DreamTask/
│
├── commands.ts           # Slash command registry (getCommands)
├── commands/             # Command implementations (80+ commands)
│   ├── commit.ts
│   ├── review.ts
│   ├── config/
│   ├── mcp/
│   └── ...
│
├── state/                # State management
│   ├── store.ts          #   Generic store (createStore)
│   ├── AppStateStore.ts  #   AppState type & defaults
│   ├── AppState.tsx      #   React context provider
│   ├── onChangeAppState.ts # State change side-effects
│   └── selectors.ts      #   State selectors
│
├── bootstrap/
│   └── state.ts          # Global mutable bootstrap state
│
├── services/             # Service layer
│   ├── api/              #   Anthropic API (client, claude, errors, retry)
│   ├── mcp/              #   MCP protocol (client, config, auth)
│   ├── analytics/        #   Analytics (GrowthBook, Datadog, events)
│   ├── compact/          #   Context compaction
│   ├── lsp/              #   Language server protocol
│   ├── oauth/            #   OAuth authentication
│   ├── plugins/          #   Plugin management
│   ├── policyLimits/     #   Enterprise policy limits
│   ├── remoteManagedSettings/ # Remote settings
│   ├── tips/             #   Contextual tips
│   └── ...
│
├── bridge/               # Remote bridge system (30+ files)
│   ├── bridgeMain.ts     #   Main bridge loop
│   ├── bridgeApi.ts      #   Bridge HTTP client
│   ├── sessionRunner.ts  #   Session process management
│   └── ...
│
├── remote/               # Remote session management
├── server/               # Direct-connect server
│
├── screens/              # Top-level screens
│   ├── REPL.tsx          #   Main interactive REPL
│   ├── Doctor.tsx        #   Diagnostics
│   └── ResumeConversation.tsx
│
├── components/           # React components (180+ files)
│   ├── App.tsx
│   ├── Messages.tsx
│   ├── PromptInput/
│   ├── design-system/    #   Theme, Box, Text, color
│   ├── permissions/
│   ├── messages/
│   ├── shell/
│   ├── TrustDialog/
│   └── ...
│
├── ink/                  # Custom Ink fork (React terminal renderer)
│   ├── root.ts
│   ├── reconciler.ts
│   ├── dom.ts
│   ├── components/
│   ├── hooks/
│   ├── events/
│   ├── layout/
│   └── termio/
│
├── ink.ts                # Ink re-exports with ThemeProvider wrapping
│
├── hooks/                # Application React hooks (90+ files)
│   ├── useCanUseTool.tsx
│   ├── useTextInput.ts
│   ├── useVoice.ts
│   └── ...
│
├── context/              # React contexts
│   ├── modalContext.tsx
│   ├── notifications.tsx
│   └── ...
│
├── plugins/              # Plugin system
│   ├── builtinPlugins.ts
│   └── bundled/
│
├── skills/               # Skill system
│   ├── bundledSkills.ts
│   ├── loadSkillsDir.ts
│   └── bundled/
│
├── keybindings/          # Keybinding system
├── vim/                  # Vim mode
├── memdir/               # Memory directory system
├── buddy/                # Companion sprite system
├── coordinator/          # Multi-agent coordinator
├── assistant/            # Assistant (Kairos) mode
├── voice/                # Voice input
│
├── context.ts            # System/user context building (git status, etc.)
├── cost-tracker.ts       # Cost & usage tracking
├── costHook.ts           # Cost threshold hook
├── history.ts            # Input history management
├── projectOnboardingState.ts # First-run onboarding
│
├── types/                # Shared type definitions
│   ├── message.ts
│   ├── permissions.ts
│   ├── hooks.ts
│   ├── ids.ts
│   └── ...
│
├── constants/            # Application constants
├── schemas/              # Validation schemas (hooks)
├── migrations/           # Data/config migrations
├── outputStyles/         # Custom output style loading
├── cli/                  # CLI output layer (print, SDK, structured IO)
├── utils/                # Utility library (300+ files)
│   ├── model/            #   Model selection & providers
│   ├── permissions/      #   Permission system
│   ├── settings/         #   Settings management
│   ├── plugins/          #   Plugin loading
│   ├── hooks/            #   Hook system
│   ├── auth.ts           #   Authentication
│   ├── config.ts         #   Configuration
│   ├── git.ts            #   Git operations
│   ├── Shell.ts          #   Shell execution
│   └── ...
│
├── native-ts/            # Native TypeScript modules
├── moreright/            # Extension modules
└── upstreamproxy/        # HTTP proxy configuration
```

* * *

## Key Architectural Patterns

### 1\. Feature Flags & Dead Code Elimination

```typescript
import { feature } from 'bun:bundle'
const module = feature('FEATURE_NAME') ? require('./module.js') : null
```

Bun's bundler eliminates entire code paths at build time for external builds. Internal-only features (COORDINATOR\_MODE, KAIROS, BRIDGE\_MODE, VOICE\_MODE, etc.) are fully stripped.

### 2\. Lazy Imports for Circular Dependency Breaking

```typescript
const getModule = () => require('./module.js') as typeof import('./module.js')
```

Heavy modules (React, agent tools, teammates) are lazily required to avoid circular dependencies and reduce startup time.

### 3\. Startup Performance Optimization

-   Side-effect imports at the top of `main.tsx` fire parallel work (MDM reads, keychain prefetch) before module evaluation completes.
-   MDM settings, OAuth, and API preconnection all run concurrently.
-   `profileCheckpoint()` marks timing milestones throughout boot.

### 4\. Dual-Path Architecture

The codebase supports two primary execution paths:

-   **Interactive (REPL)**: `main.tsx` → `setup.ts` → `launchRepl()` → `REPL.tsx` → `query.ts`
-   **Headless (SDK/Print)**: `main.tsx` → `setup.ts` → `QueryEngine` → `submitMessage()`

Both paths share the same tool, command, and API infrastructure but differ in UI rendering and state management.

### 5\. Immutable State with Functional Updates

```typescript
store.setState((prev: AppState) => ({ ...prev, field: newValue }))
```

State is `DeepImmutable` by type. Updates use functional updaters. The store notifies subscribers on reference changes only.

### 6\. MCP Integration

MCP (Model Context Protocol) is a first-class integration — tools, commands, and resources from MCP servers are dynamically merged into the available tool/command sets at runtime.
