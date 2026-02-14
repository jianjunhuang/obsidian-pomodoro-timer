# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

- `npm run dev` - Build in watch mode for development (outputs to `main.js`)
- `npm run build` - Production build with type checking
- `npm run version` - Bump version in manifest.json and versions.json (run after version change)

## Architecture Overview

This is an Obsidian plugin that provides a Pomodoro timer with task tracking capabilities. The codebase uses:
- **Svelte stores** for reactive state management across components
- **Web Worker** for accurate timer ticks via `requestAnimationFrame`
- **Task serialization** abstraction supporting both Tasks plugin and Dataview formats
- **Templater integration** for custom log formats

### Core Components

**main.ts** (`PomodoroTimerPlugin`)
- Plugin entry point that initializes Timer, Tasks, and TaskTracker
- Registers ribbon icon, status bar, commands, and the TimerView

**Timer.ts**
- Core timer logic using Svelte stores for reactive state
- Manages WORK/BREAK modes, elapsed time, and session state
- Uses `clock.worker.ts` for precise timing via Web Worker
- Handles notifications and logs sessions via Logger

**Tasks.ts & TaskTracker.ts**
- `Tasks`: Parses and caches tasks from active file using metadata cache
- `TaskTracker`: Tracks the "focused" task, manages block IDs, and updates actual pomodoro counts
- Both react to file changes and active leaf changes

**Logger.ts**
- Logs completed sessions to daily notes, weekly notes, or custom files
- Supports SIMPLE, VERBOSE, and CUSTOM (Templater) log formats
- Resolves log file based on settings (focused file, daily/weekly note, or custom path)

**Settings.ts** (`PomodoroSettings`)
- Manages plugin settings as a Svelte store
- Settings persist to `data.json` and trigger timer reconfiguration

### Serializer Pattern

The `src/serializer/` directory abstracts task parsing logic:
- `TaskDeserializer` interface for parsing task lines
- `DefaultTaskSerializer` for Tasks emoji format
- `DataviewTaskSerializer` for Dataview inline fields
- `POMODORO_REGEX` pattern: `(?:(?=[^\]]+\])\[|(?=[^)]+\))\( *🍅:: *(\d* *\/? *\d*) *)[)\]]`

### Build System

**esbuild.config.mjs**
- Bundles TypeScript and Svelte components into `main.js`
- Custom `inlineWorkerPlugin` inlines `clock.worker.ts` as a Blob URL
- Externalizes Obsidian-provided modules (obsidian, electron, codemirror, lezer)

**version-bump.mjs**
- Synchronizes version across `manifest.json`, `versions.json`, and `package.json`
- Run via `npm version` to trigger

### Svelte Components

All UI components are `.svelte` files that subscribe to stores:
- `StatusBarComponent.svelte` - Status bar timer display
- `TimerViewComponent.svelte` - Main timer panel UI
- `TasksComponent.svelte` - Task list for selecting focus task
- `TaskItemComponent.svelte` - Individual task item
- `TimerSettingsComponent.svelte` - Settings UI fragment

## Important Implementation Details

1. **Module Resolution** - Source imports omit `./src/` prefix (e.g., `import Timer from 'Timer'`) due to `baseUrl: "./src"` in tsconfig.json

2. **Timer Accuracy** - The Web Worker (`clock.worker.ts`) uses `requestAnimationFrame` for accurate timing. Low FPS mode (`lowFps` setting) reduces CPU usage by posting messages once per second instead of every frame.

3. **Task Block IDs** - When task tracking is enabled and a task is activated, the plugin automatically appends a block ID (e.g., ` ^abc123`) if one doesn't exist. This enables reliable task updates.

4. **Log Resolution Priority**:
   - If `logFocused` is enabled and task has a path → log to that file
   - Otherwise, use `logFile` setting (DAILY/WEEKLY/FILE/NONE)

5. **State Updates** - All state mutations use Svelte's `store.update()` pattern to ensure reactivity across components.

## Settings Reference

Key settings that affect behavior:
- `workLen` / `breakLen` - Session duration in minutes
- `autostart` - Auto-start next session after completion
- `enableTaskTracking` - Enable task focus and pomodoro counting
- `taskFormat` - 'TASKS' (emoji) or 'DATAVIEW' format
- `logFile` / `logLevel` / `logFormat` - Session logging configuration
- `lowFps` - Reduce animation frequency for CPU savings
