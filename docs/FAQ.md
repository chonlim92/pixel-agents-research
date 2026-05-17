# Pixel Agents — Frequently Asked Questions

**Author: Chong Kiat Lim**

---

## Agents & Multi-Agent

### Can I have multiple agents running at the same time?

**Yes.** There is no hard-coded limit on the number of agents. Each click of the `+Agent` button creates a new Claude Code terminal and a new pixel character. Additionally:

- **External sessions** (e.g., Claude Code running in another VS Code panel) are auto-detected via hooks and appear as additional characters.
- **Agent Teams** — when a lead agent spawns teammates via the `Agent` tool with `run_in_background: true`, each teammate appears as its own independent character.
- **Ephemeral subagents** (from the `Task` tool or regular `Agent` calls) show as small child characters near the parent and disappear when the tool call completes.

Each Claude Code **terminal** maps to exactly **one lead agent**, but that lead can spawn teammates that each get their own character.

**Practical limits:**

- The first 6 agents get unique skin palettes; beyond that, skins repeat with hue shifts.
- Characters need seats in the pixel office — those without a free seat spawn at random walkable tiles.
- Bounded by office layout space and screen real estate, not by code.

### How do I stop or kill a running agent?

Three ways:

1. **Click the "×" button in the office view** — select the agent character, then click the close button in the overlay bar next to its name.
2. **Click "✕" in the Debug View** — each agent card has its own close button.
3. **Kill the terminal** — close the `Pixel Agents #N` terminal in VS Code's terminal panel, or type `/exit` / press `Ctrl+C` inside it.

All three trigger the same cleanup: JSONL polling stops, file watchers are cleared, the character plays a despawn animation (matrix effect), and the seat is freed.

**Teammates are automatically removed** when their lead agent is killed.

### I killed the Claude terminal, but the agent character is still in the office. Why?

This happens when the agent was detected as an **external session** rather than a Pixel Agents managed terminal. There are two types of agents:

| Type | Created by | `terminalRef` | Removed on terminal close? |
|---|---|---|---|
| **Terminal-backed** | `+Agent` button | Set to terminal instance | **Yes** — automatically |
| **External** | Hooks / "Watch All Sessions" scanning | `undefined` | **No** — no terminal to match |

External agents (e.g., from the Claude Code VS Code extension panel or an external CLI session) have no terminal reference, so closing the terminal doesn't trigger cleanup.

**How to remove it:**
- Click the agent in the office → click the **"×"** close button in the overlay bar.

**How to prevent it:**
- Enable **hooks** (Settings → "Hooks enabled"). With hooks, Claude Code sends a `SessionEnd` event when the session ends, and external agents are removed automatically.
- Without hooks, the only auto-cleanup is a 30-second stale check that requires the JSONL file to be physically deleted from disk — which may not happen immediately.

### What happens if Claude Code crashes mid-session?

The character does **not** instantly disappear. The extension detects agent status via JSONL file polling (500ms) and hook events. With hooks enabled, a `SessionEnd` event triggers cleanup. Without hooks, heuristic timers (5s text-idle, 7s permission-wait) will eventually mark the agent as waiting. The character remains in the office until the terminal is closed or you manually remove it via the × button.

Agent state is persisted in VS Code's `workspaceState` and restored across reloads by matching to live terminals.

### Does it work with AI tools other than Claude Code (Copilot, Cursor, Aider)?

**Not yet.** Currently only Anthropic's Claude Code CLI is supported. The codebase has a `providers/` directory suggesting multi-provider support is being designed, and the roadmap mentions adapters for Codex, OpenCode, Gemini, Cursor, and Copilot — but none are implemented yet.

### Can I run it alongside the official Claude Code VS Code extension?

**Yes.** Enable **Settings → "Watch All Sessions"** (off by default) to monitor Claude Code sessions started from outside Pixel Agents — including from the Claude Code extension panel or external terminals. These appear as "external" agents that can be observed but not directly controlled (no terminal reference).

---

## Layout & Display

### The Pixel Agents tab and the Terminal tab are both in the bottom panel. How do I view them side by side?

Since the view is registered in VS Code's bottom panel, it shares space with the Terminal. Three solutions:

1. **Move to the secondary side bar (recommended)** — right-click the Pixel Agents tab → **Move View to Side Panel**. This gives it a dedicated space on the right side while keeping the terminal at the bottom. Toggle with `Ctrl+Alt+B`.
2. **Drag into the editor area** — drag the Pixel Agents tab into the editor region. It becomes a regular editor tab you can split alongside code.
3. **Split the bottom panel** — drag the Pixel Agents tab to the right edge of the bottom panel. VS Code splits the panel horizontally: terminal on the left, pixel office on the right.

### Can I customize the office layout?

**Yes.** Click the **"Layout"** button in the bottom toolbar to enter the editor. Available tools:

| Tool | Function |
|---|---|
| **Select** | Click/drag furniture |
| **Paint** | Paint floor tiles (7 patterns, HSB color) or walls (auto-tiling) |
| **Erase** | Remove tiles or furniture |
| **Place** | Place furniture from the catalog (desks, chairs, electronics, decor) |
| **Eyedropper** | Sample existing tiles or furniture |

Keyboard shortcuts: `R` (rotate), `T` (toggle state), `Delete` (remove), `Ctrl+Z` / `Ctrl+Y` (50-level undo/redo). The grid is expandable up to 64×64 tiles. Layouts can be exported/imported as JSON via the Settings modal.

### Can I use custom character skins or sprites?

**No.** Custom character sprites are not supported — only the 6 built-in diverse skins (based on [JIK-A-4 Metro City](https://jik-a-4.itch.io/metrocity-free-topdown-character-pack)) are available. Beyond 6 agents, skins repeat with random hue shifts. Custom **furniture** is supported via external asset directories, but characters are not customizable.

---

## Performance & Overhead

### Does Pixel Agents consume extra API tokens or slow down Claude Code?

**No extra API tokens.** Pixel Agents is purely observational — it reads Claude Code's JSONL transcript files and optionally receives hook events. It never injects prompts, makes API calls, or modifies Claude Code's behavior.

The only overhead is:

- JSONL file polling every 500ms per agent
- A lightweight localhost-only HTTP server for hook events
- The webview's Canvas 2D game loop at display refresh rate

This is negligible and has no effect on Claude Code's performance or token usage.

---

## Hooks & Notifications

### What are hooks and should I enable them?

Hooks are the **recommended** way for Pixel Agents to receive real-time events from Claude Code. When enabled, the extension:

1. Copies `claude-hook.js` to `~/.pixel-agents/hooks/`
2. Adds entries to `~/.claude/settings.json` for 11 event types (SessionStart, Stop, PermissionRequest, etc.)
3. Claude Code runs the hook script on each event, which POSTs to the local HTTP server

**Benefits over the fallback (JSONL-only) mode:**

- Instant permission-request detection (vs. 7-second heuristic guess)
- Accurate session start/end detection
- External session auto-discovery
- Team/teammate awareness

To disable, toggle off **Settings → "Hooks enabled"** — this removes all hook entries from `~/.claude/settings.json`.

### How do the speech bubble notifications work?

| Bubble | Meaning | Detection |
|---|---|---|
| Amber "..." | Agent is waiting for permission approval | `PermissionRequest` hook event (instant), or 7s no-data heuristic (fallback) |
| Green "✓" | Agent finished its turn | `turn_duration` JSONL record or `Stop` hook event |

Clicking a permission bubble focuses that agent's terminal so you can approve or deny.

### What about sound notifications?

Two synthesized sounds via Web Audio API (no audio files needed):

- **Done sound:** Ascending two-note chime (E5 → B5) when an agent finishes its turn
- **Permission sound:** Descending two-note tap (A5 → E5) when an agent needs approval

Toggle via **Settings → "Sound enabled"** (default: on). Note: the browser's AudioContext requires an initial user click in the webview to unlock.

---

## Setup & Compatibility

### What VS Code version is required?

**VS Code 1.105.0 or later**, as specified in `package.json`:

```json
"engines": { "vscode": "^1.105.0" }
```

### Does it work in Remote/WSL/SSH sessions?

**Not officially supported.** The extension relies on local filesystem access to `~/.claude/projects/` for JSONL files and `~/.pixel-agents/` for config, plus a localhost HTTP server on `127.0.0.1`. These mechanisms would likely not work in VS Code Remote scenarios where the extension runs on the remote host but the webview renders locally.

### Where is my data stored?

| Data | Location | Scope |
|---|---|---|
| Office layout | `~/.pixel-agents/layout.json` | User-level, shared across all VS Code windows |
| Config (asset dirs) | `~/.pixel-agents/config.json` | User-level |
| Server discovery | `~/.pixel-agents/server.json` | User-level (port, PID, auth token) |
| Hook script | `~/.pixel-agents/hooks/claude-hook.js` | User-level |
| Settings (sound, hooks, labels) | VS Code `globalState` | Per-user, survives across workspaces |
| Agent state (sessions, palettes, seats) | VS Code `workspaceState` | Per-workspace |

All file writes use atomic tmp+rename to prevent corruption. Layouts sync across VS Code windows via `fs.watch` + 2-second polling.

### How do I fully uninstall the hooks?

1. Toggle off **Settings → "Hooks enabled"** in the Pixel Agents panel — this removes all hook entries from `~/.claude/settings.json`.
2. Optionally delete the leftover script: `~/.pixel-agents/hooks/claude-hook.js`.

The hook script file is left on disk when you disable hooks, but it is no longer referenced by Claude Code.

---

## Camera & Navigation

### How do I pan and zoom the office view?

| Action | Control |
|---|---|
| **Pan** | Mouse wheel / trackpad scroll, or middle-mouse drag |
| **Zoom in/out** | `Ctrl+Scroll` (or `Cmd+Scroll` on Mac), or use the +/− buttons in the top-left corner |
| **Follow an agent** | Click an agent to select it — the camera smoothly tracks it. Click again to stop following. Any manual pan also breaks follow mode. |

Zoom range is 1×–10×. Pan is clamped so the map edge stays within view.

### Can I resize or scroll the office grid?

- **Scroll/pan:** Yes — see camera controls above.
- **Expand the grid:** In the layout editor, paint a floor or wall tile at the grid border (a ghost border appears) to expand it. Maximum grid size is **64×64 tiles** (default is 20×11).
- **Canvas auto-resizes** to fill its container and is DPR-aware (device pixel ratio).

---

## Characters & Animations

### What do the different character animations mean?

| State | Animation | Triggers |
|---|---|---|
| **Typing** | Two-frame keyboard animation (0.3s per frame) | Agent is active and using Write, Edit, or Bash tools |
| **Reading** | Book/document holding pose | Agent is using Read, Grep, Glob, WebFetch, or WebSearch tools |
| **Walking** | Four-frame walk cycle (0.15s per frame) | Moving to a seat, wandering while idle, or manually directed via right-click |
| **Idle** | Static standing pose | Agent is inactive — alternates between random wandering and resting at their seat |

### Which Claude Code tools are visualized and how?

| Tool | Status Text | Animation |
|---|---|---|
| Read | "Reading" | Reading pose |
| Grep | "Searching" | Reading pose |
| Glob | "Globbing" | Reading pose |
| WebFetch | "Fetching" | Reading pose |
| WebSearch | "Searching web" | Reading pose |
| Write | "Writing" | Typing animation |
| Edit | "Editing" | Typing animation |
| Bash | "Running" | Typing animation |
| Task / Agent | "Task" | Spawns a sub-agent character |

Unrecognized tools default to the typing animation.

### Can I assign an agent to a specific desk?

**Yes.** Click an agent to select it, then click an available (unoccupied) seat. The agent pathfinds to the new seat and the assignment is persisted across restarts. Sub-agents cannot be reassigned.

### Can I give agents custom names?

**Not manually.** Agents are automatically labeled `Agent #N`. In multi-root workspaces, agents also show the folder name. Team agents show team role labels (lead in gold, teammates in blue). You can toggle **Settings → "Always Show Labels"** to keep labels visible at all times vs. only on hover/select. There is no UI for arbitrary custom names.

### How does pathfinding work?

BFS on a 4-connected grid (up/down/left/right — no diagonals). Floor tiles are walkable; wall, void, and furniture-blocked tiles are not. Characters move at 48 pixels/sec, interpolating smoothly between tile centers. A character's own seat is temporarily unblocked during pathfinding so it can sit down.

---

## Custom Assets & Sharing

### Can I add custom furniture or asset packs?

**Yes.** Go to **Settings → "Add Asset Directory"** and point to a folder structured as:

```
my-assets/
  assets/
    furniture/
      ITEM_NAME/
        manifest.json
        sprite.png
```

Each furniture item needs a `manifest.json` defining id, name, category, sprite dimensions, footprint, and placement rules. Features include rotation groups (2-way/4-way), animation, surface placement (items on desks), wall placement, and mirror variants. External asset IDs override bundled assets on collision. Config is stored in `~/.pixel-agents/config.json`.

### How do I share my office layout with others?

Via **Settings → Export Layout** / **Import Layout**:

1. **Export:** Opens a save dialog and writes the layout as a JSON file (default: `pixel-agents-layout.json`).
2. **Import:** Opens a file picker, validates the JSON, and applies it immediately.

Send the exported `.json` file to others — they import through the same menu.

### Can I take a screenshot of my office?

**No.** There is no built-in screenshot or image export feature. Use your OS screenshot tool (e.g., `Win+Shift+S` on Windows, `Cmd+Shift+4` on Mac).

---

## Multi-Window & Persistence

### Does the office persist across VS Code restarts?

**Yes, both layout and agent state persist:**

| Data | Survives Restart? | Details |
|---|---|---|
| Office layout | Yes | Saved to `~/.pixel-agents/layout.json` via atomic writes |
| Agent state | Partially | Persisted to workspace state; only restored if the agent's terminal is still alive |
| Seat assignments | Yes | Stored per-agent with palette and seat ID |
| Settings | Yes | Stored in VS Code `globalState` |

### I renovated my pixel office layout. Will it persist next time I reopen VS Code? What about across different projects?

**Yes to both.** Your office layout is saved to `~/.pixel-agents/layout.json` — a **user-level** file, not workspace-specific. This means:

- **Restarts:** The layout survives VS Code restarts, updates, and window closures. Every change is saved immediately via atomic writes (tmp file + rename) to prevent corruption.
- **Across projects:** The same layout is shared across **all** workspaces and projects. Open a different project folder and you'll see the same office you designed.
- **Across windows:** If you have multiple VS Code windows open, layout changes in one sync to others within ~2 seconds via file watching.

The only thing that does **not** carry across projects is agent state (which agents are running, their seat assignments) — that is per-workspace since agents are tied to terminals in each window.

### What happens when I open multiple VS Code windows?

- **Layout is shared** — all windows read/write the same `~/.pixel-agents/layout.json`. Changes in one window auto-sync to others within ~2 seconds.
- **Agents are per-window** — each window manages its own agents independently (tied to terminals in that window).
- **Server is singleton** — only one window runs the HTTP hook server. Other windows detect the running server via `~/.pixel-agents/server.json` and reuse it.

---

## Debugging & Diagnostics

### What is the Debug View?

A diagnostic overlay (accessible from the toolbar) showing per-agent cards with:

- **Agent ID** and close button
- **Active tool list** with color-coded dots: green (active), amber (permission wait), gray (done). Sub-agent tools shown indented.
- **Waiting indicator** when the agent appears stuck with no active tools
- **Connection diagnostics** (refreshed every 2s): JSONL file status, lines processed, last data timestamp, file path, and warnings if data isn't being parsed

### How does the HTTP server work? Is it secure?

- **Localhost only:** Binds to `127.0.0.1` on a random OS-assigned port. Never exposed to the network.
- **Auth token:** Random UUID generated per server start. All hook requests require `Authorization: Bearer <token>` validated with `crypto.timingSafeEqual()` (timing-attack resistant).
- **Body limits:** 64KB max request body, 5-second request timeout.
- **Discovery:** Port, PID, and token written to `~/.pixel-agents/server.json` so hook scripts can locate the server. This file is deleted when the owning window closes.

---

## Keyboard Shortcuts

### What keyboard shortcuts are available?

**In the office view (normal mode):** No keyboard shortcuts — all interaction is mouse-based (click, scroll, middle-drag).

**In the layout editor:**

| Key | Action |
|---|---|
| `Escape` | Deselect (multi-stage: catalog item → tool → furniture → close editor) |
| `Delete` / `Backspace` | Delete selected furniture |
| `R` | Rotate selected furniture |
| `T` | Toggle furniture state (on/off) |
| `Ctrl+Z` | Undo (up to 50 levels) |
| `Ctrl+Y` / `Ctrl+Shift+Z` | Redo |
