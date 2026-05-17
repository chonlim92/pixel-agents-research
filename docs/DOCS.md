# Pixel Agents — Complete Technical Documentation

**Author: Chong Kiat Lim**

> A VS Code extension that turns Claude Code AI agents into animated pixel art characters in a virtual office.

![Pixel Agents Demo](../images/pixel-agents-demo.png)
*Multi-agent office in action — two agents researching in parallel with speech bubbles showing live status*

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [How It Works — End to End](#how-it-works--end-to-end)
4. [Installation & Setup](#installation--setup)
5. [Usage Guide](#usage-guide)
6. [Agent Detection & Status Tracking](#agent-detection--status-tracking)
7. [The Pixel Office](#the-pixel-office)
8. [Character System](#character-system)
9. [Layout Editor](#layout-editor)
10. [Hooks System](#hooks-system)
11. [Server Component](#server-component)
12. [Asset System](#asset-system)
13. [Configuration & Persistence](#configuration--persistence)
14. [Known Limitations](#known-limitations)
15. [Troubleshooting](#troubleshooting)
16. [Tech Stack](#tech-stack)

---

## Overview

**Pixel Agents** is a VS Code extension that provides a visual, game-like interface for managing multiple Claude Code AI agents. Each agent running in a VS Code terminal is represented as an animated pixel art character inside a customizable virtual office. The characters walk, sit at desks, type, read, and visually reflect what their corresponding AI agent is actually doing in real time.

### What Problem Does It Solve?

When running multiple Claude Code agents simultaneously (e.g., one fixing bugs, another writing tests, a third refactoring), it becomes difficult to track which agent is doing what, which one needs attention, and which one has finished. Pixel Agents solves this by giving you a single visual dashboard — a pixel art office — where you can see all agents at a glance.

### Key Capabilities

| Capability | Description |
|---|---|
| **One agent, one character** | Every Claude Code terminal gets its own animated pixel character |
| **Live activity tracking** | Characters animate based on real tool usage (writing code, reading files, running bash commands) |
| **Speech bubbles** | Visual indicators when an agent is waiting or needs permission approval |
| **Sound notifications** | Optional audio chimes when agents complete turns or need attention |
| **Sub-agent visualization** | `Task` tool sub-agents spawn as separate characters linked to their parent |
| **Office layout editor** | Full drag-and-drop editor for floors, walls, and furniture |
| **Persistent layouts** | Office design shared across all VS Code windows |
| **Custom asset packs** | Load third-party or custom furniture from external directories |
| **6 diverse character skins** | With automatic hue-shifting for uniqueness beyond 6 agents |

---

## Architecture

Pixel Agents has a four-layer architecture:

```
┌──────────────────────────────────────────────────────────────────┐
│                        Webview UI                                │
│  React 19 + Canvas 2D + Vite                                    │
│  (webview-ui/src/)                                               │
│  Game loop, sprite rendering, character FSM, layout editor       │
├──────────────────────────────────────────────────────────────────┤
│                   Extension Backend                              │
│  Node.js + VS Code API + esbuild                                 │
│  (src/)                                                          │
│  Agent lifecycle, JSONL parsing, file watching, asset loading    │
├──────────────────────────────────────────────────────────────────┤
│                   Server Component                               │
│  HTTP server on 127.0.0.1 (localhost only)                       │
│  (server/src/)                                                   │
│  Receives Claude Code Hook events for instant status updates     │
├──────────────────────────────────────────────────────────────────┤
│                     Shared Module                                │
│  PNG decoding, manifest parsing, constants                       │
│  (shared/assets/)                                                │
│  Platform-independent code used by both backend and frontend     │
└──────────────────────────────────────────────────────────────────┘
```

### Communication Flow

```
Claude Code CLI ──(JSONL files)──→ Extension Backend ──(postMessage)──→ Webview UI
                 ──(HTTP hooks)──→ Server ──(callback)──→ Extension Backend
```

- **Extension ↔ Webview**: VS Code's `postMessage` protocol (bidirectional)
- **Claude Code → Extension**: JSONL transcript file polling (500ms) + HTTP hook events (instant)
- **Server → Extension**: Direct callback functions (server runs in the same Node.js process)

### Module Breakdown

| Module | Path | Responsibility |
|---|---|---|
| `extension.ts` | `src/` | Entry point — registers view provider and commands |
| `PixelAgentsViewProvider.ts` | `src/` | Central orchestrator — manages webview, agents, assets, server |
| `agentManager.ts` | `src/` | Terminal lifecycle — launch, remove, restore, persist agents |
| `transcriptParser.ts` | `src/` | JSONL parsing — extracts tool usage, status, tokens from transcripts |
| `fileWatcher.ts` | `src/` | File system monitoring — polls JSONL files, detects new sessions |
| `timerManager.ts` | `src/` | Timer logic — waiting (5s) and permission (7s) detection |
| `assetLoader.ts` | `src/` | PNG/manifest loading — characters, floors, walls, furniture |
| `layoutPersistence.ts` | `src/` | Layout file I/O — atomic writes, cross-window sync |
| `configPersistence.ts` | `src/` | Config file I/O — external asset directories |
| `server.ts` | `server/src/` | HTTP server — receives hook events from Claude Code |
| `hookEventHandler.ts` | `server/src/` | Event routing — maps hook events to agent actions |
| `App.tsx` | `webview-ui/src/` | React composition root — hooks + components |
| `officeState.ts` | `webview-ui/src/office/engine/` | Game world state — characters, seats, furniture |
| `characters.ts` | `webview-ui/src/office/engine/` | Character FSM — idle/walk/type state machine |
| `renderer.ts` | `webview-ui/src/office/engine/` | Canvas 2D renderer — z-sorted pixel-perfect rendering |
| `gameLoop.ts` | `webview-ui/src/office/engine/` | `requestAnimationFrame` loop with delta time |

---

## How It Works — End to End

Here is the complete data flow from clicking "+ Agent" to seeing a character type at its desk:

### Step 1: Agent Creation

1. User clicks **"+ Agent"** in the bottom toolbar
2. Webview sends `{ type: 'openClaude' }` message to the extension
3. Extension creates a new VS Code terminal and sends `claude --session-id <uuid>`
4. An `AgentState` object is created with the session UUID
5. Extension sends `agentCreated` message to the webview
6. If hooks are enabled, the extension registers the agent with the hook event handler

### Step 2: Character Spawning

1. Webview receives `agentCreated`, calls `officeState.addAgent()`
2. A diverse character palette is selected (ensures first 6 agents get unique skins)
3. A free seat is found (preferring seats facing electronic furniture like monitors)
4. The character spawns at the seat position with a **Matrix-style digital rain animation** (0.3s)
5. After the spawn effect completes, the character enters the IDLE state

### Step 3: Session Detection

1. Extension polls every 1 second for the JSONL file at `~/.claude/projects/<project-hash>/<session-id>.jsonl`
2. The project hash is derived from the workspace path (with special characters replaced by `-`)
3. Once the file appears, a 500ms file watcher begins reading new lines

### Step 4: Activity Tracking

1. When Claude uses a tool (e.g., writing a file), a new JSONL line appears:
   ```json
   {"type":"assistant","message":{"content":[{"type":"tool_use","id":"toolu_xxx","name":"Write","input":{"file_path":"main.ts"}}]}}
   ```
2. The extension's transcript parser extracts the tool name and generates a human-readable status: `"Writing main.ts"`
3. Extension sends `agentToolStart` to the webview
4. A 7-second permission timer starts (in case the tool gets stuck waiting for approval)

### Step 5: Character Animation

1. Webview receives `agentToolStart`, marks the character as active
2. Character FSM transitions: **IDLE → WALK** (BFS pathfinding to assigned seat) **→ TYPE**
3. The animation type depends on the tool:
   - **Typing animation** (2 frames): Write, Edit, Bash, Task
   - **Reading animation** (2 frames): Read, Grep, Glob, WebFetch, WebSearch
4. The `ToolOverlay` component renders the status text (`"Writing main.ts"`) above the character

### Step 6: Completion & Waiting

1. When the tool completes, a `tool_result` record appears in the JSONL
2. After a 300ms debounce (prevents flicker), `agentToolDone` is sent to the webview
3. When the turn ends (detected via `turn_duration` system record), the agent transitions to "waiting"
4. Character shows a green checkmark bubble, plays an ascending chime
5. Character returns to IDLE, starts wandering randomly around the office

### Step 7: Permission Needed

1. If 7 seconds pass after a tool starts with no new JSONL data, the extension infers the tool needs permission
2. An amber "..." speech bubble appears above the character
3. A descending attention sound plays
4. The bubble persists until the user clicks it (which focuses the agent's terminal)

---

## Installation & Setup

### Prerequisites

- **VS Code** 1.105.0 or later
- **Claude Code CLI** installed and configured (`claude` command available in PATH)
- **Node.js** (for building from source)
- **Platform**: Windows, Linux, and macOS supported

### Install from VS Code Marketplace (Recommended)

1. Open VS Code
2. Go to Extensions (`Ctrl+Shift+X`)
3. Search for "Pixel Agents"
4. Click Install

### Install from Source

```bash
git clone https://github.com/pablodelucca/pixel-agents.git
cd pixel-agents
npm install
cd webview-ui && npm install && cd ..
npm run build
```

Then press **F5** in VS Code to launch the Extension Development Host with the extension loaded.

### Build Commands

| Command | Purpose |
|---|---|
| `npm run build` | Full build (type check + lint + esbuild + webview build) |
| `npm run watch` | Watch mode for development (esbuild + tsc in parallel) |
| `npm run test` | Run all tests (webview + server) |
| `npm run lint` | Lint all source files |
| `npm run format` | Format all source files with Prettier |
| `npm run e2e` | Run Playwright end-to-end tests |

---

## Usage Guide

### Basic Usage

1. **Open the Pixel Agents panel** — It appears in the bottom panel area alongside your terminal. You can also use the command palette: `Pixel Agents: Show Panel`
2. **Click "+ Agent"** — Spawns a new Claude Code terminal and its character
   - Right-click for the option to launch with `--dangerously-skip-permissions` (bypasses all tool approval prompts)
3. **Interact with Claude** — Start giving prompts to Claude in the terminal. Watch the character react in real time
4. **Select a character** — Click on a character to select it. This will focus its terminal
5. **Reassign seats** — With a character selected, click on any empty seat to move it there
6. **Close an agent** — Click the X button on the tool overlay above a selected character

### Multi-Root Workspaces

If your VS Code window has multiple workspace folders, a folder selector appears in the bottom toolbar. Each agent's Claude Code instance uses the selected folder as its working directory.

### Watch All Sessions Mode

Enable "Watch All Sessions" in Settings to monitor Claude Code sessions started from outside Pixel Agents (e.g., from the Claude Code extension panel or external terminals). These appear as "external" agents that can be observed but not directly controlled.

---

## Agent Detection & Status Tracking

### JSONL Transcript Files

Claude Code stores conversation transcripts as JSONL (JSON Lines) files at:
```
~/.claude/projects/<project-hash>/<session-id>.jsonl
```

The `<project-hash>` is derived from the workspace path with special characters (`:`, `\`, `/`) replaced by `-`. The extension watches these files to track agent activity.

### JSONL Record Types

| Record Type | Content | What the Extension Does |
|---|---|---|
| `assistant` with `tool_use` | Agent calling tools | Sends `agentToolStart`, starts permission timer |
| `assistant` (text only) | Agent thinking/responding | Starts 5s text-idle timer |
| `user` with `tool_result` | Tool results returned | Sends `agentToolDone` (300ms delay) |
| `user` (text) | New user prompt | Clears all agent activity (new turn) |
| `system` + `turn_duration` | Turn ended | Reliable turn-end signal → agent waiting |
| `progress` / `agent_progress` | Sub-agent activity | Forwards tool events for sub-agent characters |
| `progress` / `bash_progress` | Long-running bash | Restarts permission timer (tool is still alive) |

### Dual-Mode Detection

The extension uses two complementary detection modes:

**Hooks Mode (Preferred)** — When enabled, Claude Code's Hooks API sends instant HTTP events to the extension's local server. This provides:
- Zero-latency tool start detection (vs 500ms polling delay)
- Exact permission detection (vs 7s heuristic timer)
- Reliable turn-end signals (vs 5s text-idle heuristic)

**Heuristic Mode (Fallback)** — When hooks are unavailable, the extension falls back to:
- 500ms JSONL file polling per agent
- 1s project directory scanning for new sessions
- 3s external session scanning
- 7s permission timer, 5s text-idle timer

Even with hooks active, JSONL polling continues for tool content (status text, animation types) that hooks don't carry.

### Tool Status Display

The extension generates human-readable status strings from tool usage:

| Tool | Example Status |
|---|---|
| `Write` | `"Writing main.ts"` |
| `Read` | `"Reading main.ts"` |
| `Edit` | `"Editing main.ts"` |
| `Bash` | `"Running: git status..."` |
| `Task` | `"Subtask: fix the bug..."` |
| `Grep` | `"Searching for 'TODO'..."` |
| `WebFetch` | `"Fetching https://..."` |
| `AskUserQuestion` | `"Waiting for your answer"` |

---

## The Pixel Office

### Rendering Engine

The office is rendered using **Canvas 2D** with pixel-perfect integer zoom. The game loop runs at the display's refresh rate using `requestAnimationFrame`, with delta time capped at 100ms to prevent physics jumps.

**Rendering pipeline:**
1. Clear canvas
2. Draw floor tiles (colorized grayscale sprites)
3. Draw wall base colors
4. Collect all entities (furniture, characters, wall 3D pieces, speech bubbles)
5. Z-sort by Y-depth (entities further down the screen render in front)
6. Draw entities in order
7. Draw editor overlays (if in edit mode)

**Zoom system:** The zoom level is always an integer (1x–10x), ensuring pixel-perfect rendering. Default zoom is `Math.round(2 × devicePixelRatio)`. The canvas backing store is set to device pixel dimensions directly — no `ctx.scale()`.

**Camera:** Pan by middle-mouse dragging. Clicking a character auto-follows it with smooth camera tracking. Follow is cleared on deselection or manual pan.

### Z-Sorting Rules

Entities are depth-sorted to create correct visual layering:
- Characters render in front of same-row furniture (desks behind them, chairs under them)
- Back-facing chairs render in front of their seated character (showing the chair back)
- Surface items on desks render in front of the desk
- Everything is sorted by `zY` (Y coordinate used for depth)

---

## Character System

### Finite State Machine

Each character runs a simple 3-state FSM:

```
                    ┌──────────────┐
     ┌──────────────│     IDLE     │──────────────┐
     │  (wander     │  (standing)  │  (isActive   │
     │   timer)     └──────┬───────┘   = true)    │
     │                     │                      │
     ▼                     │                      ▼
┌────────────┐             │             ┌────────────┐
│    WALK    │             │             │    WALK    │
│ (to random │             │             │ (to seat)  │
│   tile)    │             │             │            │
└─────┬──────┘             │             └─────┬──────┘
      │ (arrived)          │                   │ (arrived)
      ▼                    │                   ▼
┌────────────┐             │             ┌────────────┐
│    IDLE    │◄────────────┘             │    TYPE    │
│ (standing) │      (isActive            │ (typing or │
│            │       = false)            │  reading)  │
└────────────┘                           └────────────┘
```

### States

| State | Animation | Trigger |
|---|---|---|
| **IDLE** | Static standing pose | Default; after task completion; after wandering |
| **WALK** | 4-frame walk cycle | Moving to seat (when active) or wandering randomly |
| **TYPE** | 2-frame typing or reading | Seated at desk, agent is using tools |

### Wander AI

When idle, characters periodically wander around the office:
1. A random wander timer fires
2. Character picks a random walkable tile via BFS pathfinding (4-directional, no diagonals)
3. Character walks there at a set speed
4. After 3–6 wanders (random limit), character returns to its seat for a long rest (120–240 seconds)

### Sprite System

- **6 pre-colored character palettes** loaded from PNG spritesheets (`char_0.png` – `char_5.png`)
- Each spritesheet: 112×96 pixels = 7 frames × 16px wide, 3 direction rows × 32px tall
- **Directions**: DOWN, UP, RIGHT (LEFT = horizontally flipped RIGHT)
- **Frames**: walk1, walk2, walk3, type1, type2, read1, read2
- **Hue shifting**: When more than 6 agents exist, additional agents reuse palettes with random hue rotation (45°–315°)

### Speech Bubbles

| Bubble | Color | Meaning | Duration |
|---|---|---|---|
| Green checkmark (✓) | Green | Agent finished its turn, waiting for input | Auto-fades after ~2s |
| Amber ellipsis (...) | Amber | Agent likely needs permission approval | Persistent until clicked |

### Spawn/Despawn Effects

Characters appear and disappear with a **Matrix-style digital rain animation** (0.3s duration):
- **Spawn**: Green rain columns sweep top-to-bottom, revealing the character pixel-by-pixel
- **Despawn**: Rain sweep consumes the character, pixel-by-pixel
- Per-column staggering with random seeds creates an organic sweep pattern
- Hash-based flicker (~70% pixel visibility per frame) adds shimmer
- Restored agents (from session persistence) skip the spawn effect

---

## Layout Editor

### Entering Edit Mode

Click the **"Layout"** button in the bottom toolbar to toggle the editor.

### Tools

| Tool | Shortcut | Description |
|---|---|---|
| **Select** | — | Click to select furniture, drag to move |
| **Paint** | — | Paint floor tiles or walls on the grid |
| **Erase** | — | Remove tiles or furniture |
| **Place** | — | Place furniture from the catalog |
| **Eyedropper** | — | Pick a tile type by clicking on the grid |
| **Pick** | — | Pick a furniture item by clicking on it |

### Keyboard Shortcuts

| Key | Action |
|---|---|
| `R` | Rotate selected furniture |
| `T` | Toggle furniture state (e.g., monitor on/off) |
| `Delete` | Remove selected furniture |
| `Ctrl+Z` | Undo (50 levels) |
| `Ctrl+Y` | Redo |
| `Escape` | Exit tool / deselect / exit edit mode |

### Grid

- Default size: 20×11 tiles
- Maximum size: 64×64 tiles
- Expandable by clicking the ghost border outside the current grid

### Floor Types

7 grayscale floor patterns available, each with full HSB (Hue, Saturation, Brightness) color control via the Colorize module.

### Wall System

Auto-tiling walls that automatically connect with adjacent wall tiles. Walls use a 16-piece bitmask system from a 4×4 grid spritesheet. Walls also support HSB color customization.

### Furniture

Furniture items are organized into categories: desks, chairs, storage, electronics, decor, wall items, and misc.

Features:
- **Rotation groups**: Furniture can be rotated through available orientations (front/back/left/right)
- **State groups**: Some furniture has on/off states (e.g., monitors turn on when an agent sits nearby)
- **Animation**: Some furniture has animated frames
- **Mirror support**: Some items have left/right mirrored variants
- **Auto-state**: Electronics automatically switch to "ON" sprites when an active agent faces them

### Import/Export

- **Export**: Settings → Export Layout → saves as JSON file
- **Import**: Settings → Import Layout → loads from JSON file (validates `version: 1`)
- **Export as Default**: Command palette → "Pixel Agents: Export Layout as Default" (for contributors)

---

## Hooks System

### What Are Hooks?

Claude Code's Hooks API allows external tools to receive instant notifications about agent events. Pixel Agents installs a small JavaScript hook script that POSTs events to the extension's local HTTP server.

### Hook Events

| Event | Description | Action Taken |
|---|---|---|
| `SessionStart` | Agent session begins/resumes | Confirms agent registration |
| `SessionEnd` | Session exits or clears | Sets pending-clear state |
| `Stop` | Turn completed | Marks agent as waiting (reliable) |
| `PermissionRequest` | Tool needs permission | Shows permission bubble (exact) |
| `Notification` | Idle or permission notification | Routes to appropriate handler |
| `UserPromptSubmit` | User submits a prompt | Instant active state |
| `PreToolUse` | Tool about to start | Instant tool display (no polling delay) |
| `PostToolUse` | Tool completed successfully | Tool completion |
| `PostToolUseFailure` | Tool failed | Tool completion (with failure) |
| `SubagentStart` | Sub-agent spawned | Creates sub-agent character |
| `SubagentStop` | Sub-agent finished | Removes sub-agent character |

### Hook Installation

When hooks are enabled (default), the extension:
1. Writes a hook script to `~/.pixel-agents/hooks/claude-hook.js`
2. Registers it in Claude Code's settings at `~/.claude/settings.json`
3. The hook script reads event data from stdin and POSTs to the local server

### Enabling/Disabling Hooks

Toggle in Settings → "Hooks enabled". When disabled, the extension falls back to heuristic-only mode (JSONL polling).

---

## Server Component

### Purpose

The server is a lightweight HTTP server bound to `127.0.0.1` (localhost only) on a random port. Its sole purpose is to receive hook events from Claude Code's hook scripts.

### Security

- **Auth token**: Generated as a UUID, required in `Authorization: Bearer <token>` header
- **Timing-safe comparison**: Uses `crypto.timingSafeEqual()` to prevent timing attacks
- **Body size limit**: 64KB maximum
- **Localhost only**: Binds to `127.0.0.1`, not accessible from the network
- **Provider ID validation**: Route parameter validated against `^[a-z0-9-]+$`

### Multi-Window Support

Server discovery is handled via `~/.pixel-agents/server.json`:
```json
{
  "port": 12345,
  "pid": 67890,
  "token": "uuid-auth-token",
  "startedAt": "2024-01-01T00:00:00Z"
}
```

When a second VS Code window opens, it checks if the PID in `server.json` is still alive. If so, it reuses the existing server. If not, it starts a new one.

### Routes

| Route | Method | Auth | Purpose |
|---|---|---|---|
| `/api/health` | GET | No | Health check |
| `/api/hooks/:providerId` | POST | Yes | Receive hook events |

---

## Asset System

### Character Sprites

6 pre-colored character PNGs (`char_0.png` – `char_5.png`) based on [JIK-A-4 Metro City](https://jik-a-4.itch.io/metrocity-free-topdown-character-pack). Each is a 112×96 spritesheet containing 7 frames × 3 directions.

### Floor Tiles

7 grayscale floor patterns in `floors.png` (112×16). Colorized at runtime using HSB controls.

### Wall Tiles

4×4 grid of 16×32 auto-tile pieces in `walls.png` (64×128). Uses bitmask-based auto-tiling for seamless connections.

### Furniture

Each furniture item is a folder under `assets/furniture/` containing:
- One or more PNG sprite files
- A `manifest.json` declaring metadata, categories, rotation groups, state groups, and animation frames

The manifest supports nested structures that are flattened via `flattenManifest()` into a flat catalog.

### External Asset Packs

Users can load custom furniture from any directory:
1. Settings → "Add Asset Directory"
2. The directory must follow the same structure as `assets/furniture/`
3. External assets are merged with bundled assets (external IDs override on collision)

### Sprite Cache

A two-level `WeakMap` cache system:
- `SpriteData → HTMLCanvasElement` per zoom level (pixel data → rendered canvas)
- `SpriteData → SpriteData` for outline sprites (used for selection/hover highlights)

---

## Configuration & Persistence

### Settings (Global State)

| Setting | Default | Description |
|---|---|---|
| Sound enabled | `true` | Play audio notifications |
| Hooks enabled | `true` | Use Claude Code Hooks API for instant detection |
| Watch All Sessions | `false` | Monitor external Claude Code sessions |
| Always show labels | `false` | Show agent status labels always (not just on hover) |

### Agent State (Workspace State)

Agent data is persisted per-workspace in VS Code's `workspaceState`:
- Agent ID, session ID, palette, hue shift, seat assignment
- Restored on window reload by matching persisted data to live terminals

### Layout (User-Level File)

Stored at `~/.pixel-agents/layout.json`:
- Shared across all VS Code windows and workspaces
- Atomic writes via `.tmp` + rename (prevents corruption)
- Cross-window sync via hybrid `fs.watch` + 2s polling
- Own-write detection prevents re-reading self-triggered changes

### Config (User-Level File)

Stored at `~/.pixel-agents/config.json`:
- External asset directory paths
- Atomic writes via `.tmp` + rename

---

## Known Limitations

### Agent-Terminal Synchronization

The connection between agents and Claude Code terminal instances is not fully robust. It can desync when:
- Terminals are rapidly opened and closed
- Terminals are restored across VS Code session restarts
- The workspace path contains special characters that affect project hash computation
- Multiple VS Code windows target the same workspace

### Heuristic Status Detection

Without hooks, agent status detection relies on timing heuristics:
- **Permission detection**: If no new JSONL data arrives within 7 seconds of a tool starting, the agent is *assumed* to need permission. This can misfire for slow tools or network operations
- **Turn completion**: A 5-second text-idle timer assumes the turn is done if no new data arrives. This can incorrectly trigger during long thinking pauses
- **Tool done delay**: A 300ms delay before reporting tool completion prevents flicker but adds slight latency

With hooks enabled, these issues are largely eliminated — but hooks require Claude Code to support the Hooks API.

### Session File Discovery

- The project hash computation (`workspace path → dash-separated string`) must exactly match Claude Code's internal logic. Mismatches mean the extension can't find transcript files
- Case sensitivity differs between Windows (case-insensitive) and Linux/macOS (case-sensitive), requiring fuzzy matching on Windows
- Sessions started before the extension was activated may not be detected

### Visual Limitations

- **6 unique character skins**: Beyond 6 simultaneous agents, characters reuse skins with hue shifts, which can make them look similar
- **No custom character sprites**: Users cannot upload their own character designs (only furniture is customizable)
- **Canvas rendering**: The 2D canvas approach works well but doesn't support complex effects like true transparency blending or GPU acceleration
- **Grid-based movement**: Characters move on a tile grid (no smooth freeform movement), which limits pathfinding to 4-directional BFS

### Functional Limitations

- **Claude Code only**: Currently only works with Anthropic's Claude Code CLI. Other AI agents (Copilot, Codex, Cursor) are not supported (planned for future)
- **VS Code only**: Runs as a VS Code extension. Standalone Electron app, web deployment, or IDE integration for other editors are not available yet
- **No agent orchestration**: Pixel Agents is observational — you can see and monitor agents, but you can't assign tasks, redirect agents, or manage work queues through the interface
- **No model/context visibility**: You can't see which model an agent is using, its remaining context window, or detailed token usage (only aggregate input/output tokens are tracked)
- **No conversation history**: You can't view an agent's conversation history through the Pixel Agents UI — you must switch to its terminal
- **Single layout**: One office layout shared across all workspaces. You can't have different offices for different projects

### Performance

- **JSONL polling**: Even with hooks, JSONL files are polled every 500ms for content details. For many simultaneous agents, this can add up
- **Large offices**: The 64×64 tile maximum keeps performance reasonable, but heavily decorated offices with many animated furniture items may affect frame rates
- **Sprite cache**: The `WeakMap`-based cache is efficient but creates new canvases on zoom level changes

---

## Troubleshooting

### Agent doesn't spawn or stays idle

1. **Check Debug View**: Settings → toggle "Debug View". Shows per-agent diagnostics including JSONL file status, lines parsed, and last data timestamp
2. **Verify Claude Code**: Ensure `claude` CLI is installed and in your PATH. Try running `claude` directly in a terminal
3. **Check JSONL path**: Debug View shows the expected JSONL path. Verify the file exists at `~/.claude/projects/<hash>/`
4. **Check hooks**: If hooks are enabled, verify `~/.pixel-agents/server.json` exists and the PID is alive

### Characters desync from terminals

1. Close the desynced agent (X button on the character overlay)
2. Click "+ Agent" to create a fresh agent
3. If persistent, try reloading the VS Code window (`Ctrl+Shift+P` → "Developer: Reload Window")

### No sound notifications

1. Verify "Sound enabled" is checked in Settings
2. Click anywhere on the canvas (WebView AudioContext requires user interaction to start)
3. Check your system volume

### Layout not loading

1. Check if `~/.pixel-agents/layout.json` exists and is valid JSON
2. Try importing a layout from Settings → Import Layout
3. Delete `~/.pixel-agents/layout.json` to reset to the bundled default

### Debug Console (Development)

When running from source via F5, open VS Code's **View → Debug Console**. Search for `[Pixel Agents]` to see detailed logs about project directory resolution, JSONL polling, path encoding, and unrecognized record types.

---

## Tech Stack

| Component | Technology |
|---|---|
| Extension backend | TypeScript, VS Code Extension API, esbuild |
| Webview frontend | React 19, TypeScript, Vite, Canvas 2D |
| Server | Node.js HTTP (no frameworks) |
| Testing | Vitest (server), Vitest + React Testing Library (webview), Playwright (e2e) |
| Linting | ESLint with TypeScript parser |
| Formatting | Prettier |
| Build | esbuild (extension), Vite (webview) |
| Git hooks | Husky + lint-staged |

### Key Dependencies

- **React 19** — UI component framework for the webview
- **Vite** — Fast bundler/dev server for the webview
- **esbuild** — Fast bundler for the extension backend
- **pngjs** — PNG decoding for sprite loading (Node.js side)
- **Playwright** — End-to-end browser testing

---

## Future Vision

The project's long-term vision (from the roadmap) includes:

- **Agent-agnostic**: Support for Codex, OpenCode, Gemini, Cursor, Copilot via composable provider adapters
- **Platform-agnostic**: Electron app, web app, or other host environments beyond VS Code
- **Deep inspection**: Click any agent to see model, branch, system prompt, and work history
- **Desks as directories**: Drag agents to desks to assign working directories
- **Kanban integration**: Office wall with a task board where idle agents pick up work autonomously
- **Token health bars**: Context window and rate limits visualized as in-game stats
- **Custom characters**: Upload your own character sprites and themes
- **3D/VR**: Move beyond pixel art into 3D or VR environments
