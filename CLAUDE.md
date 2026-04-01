# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **source code** for Claude Code v2.1.88, extracted from the npm package `@anthropic-ai/claude-code`. It's a TypeScript/React-Ink CLI application for AI-assisted coding.

## Build Commands

```bash
# Install dependencies
npm install

# Type check (must run prepare-src first)
npm run check

# Build (transforms src/ → build-src/ → dist/)
npm run build

# Run the CLI directly
node dist/cli.js --version
node dist/cli.js -p "Hello Claude"

# Run pre-built CLI (no build needed)
node cli.js --version
```

**Note**: Full rebuild requires Bun for compile-time intrinsics (`feature()`, `MACRO`, `bun:bundle`). esbuild build is ~95% complete with ~108 feature-gated modules missing.

## Architecture

```
Entry: src/entrypoints/cli.tsx → src/query.ts (main agent loop)
         ↓
    QueryEngine.ts (SDK/headless query lifecycle)
         ↓
    ┌─────────────────────────────────────────┐
    │  Tools (40+)     │  Services  │  State  │
    │  src/tools/      │  src/services/  │  src/state/  │
    │  Tool.ts factory │             │  AppStateStore  │
    └─────────────────────────────────────────┘
```

**Core files:**
- `src/query.ts` — Main while-true agent loop, largest file (~785KB)
- `src/Tool.ts` — Tool interface + `buildTool` factory
- `src/tools.ts` — Tool registry, presets, filtering
- `src/QueryEngine.ts` — SDK/headless query lifecycle engine
- `src/Task.ts` — Task types, IDs, state base
- `src/commands.ts` — Slash command definitions
- `src/context.ts` — User input context
- `src/cost-tracker.ts` — API cost accumulation

**Key patterns:**
- **Streaming**: AsyncGenerator-based streaming from API to consumer via `QueryEngine`
- **State**: `AppStateStore` with React Context integration, Zustand-like store pattern
- **Permission System**: Multi-layer (validateInput → PreToolUse Hooks → Permission Rules → Interactive Prompt → checkPermissions)
- **Feature Flags**: Bun compile-time `feature()` intrinsics — all return `false` in published source

## Directory Structure

```
src/
├── main.tsx              # REPL bootstrap (4,683 lines)
├── query.ts              # Main agent loop (~785KB)
├── Tool.ts               # Tool interface + builder
├── tools.ts              # Tool registry + filtering
├── commands.ts           # Slash command definitions
├── queryEngine.ts        # SDK query lifecycle
├── context.ts            # User input context
├── setup.ts              # First-run setup flow
├── bridge/               # Claude Desktop / remote bridge
├── cli/                  # CLI infrastructure (handlers, transports)
├── commands/              # ~80+ slash commands (agents, compact, config, help, login, mcp, memory, plan, resume, review, etc.)
├── components/            # React/Ink terminal UI (146 subdirectories)
├── entrypoints/           # Application entry points (cli.tsx, sdk/, mcp.ts)
├── hooks/                 # React hooks (87 subdirectories)
├── services/               # Business logic (API, analytics, compact, MCP, tools, plugins)
├── state/                 # Application state (AppStateStore)
├── tasks/                 # Task implementations (LocalShell, LocalAgent, RemoteAgent, InProcessTeammate, Dream)
├── tools/                 # 40+ tool implementations (Bash, FileRead, FileEdit, Glob, Grep, Agent, WebFetch, MCP, etc.)
├── screens/               # Terminal screens
├── server/                # Server components
├── skills/                # Skills system
├── migrations/            # Data migrations
├── vim/                   # Vim mode
└── voice/                 # Voice input/output
```

## Missing Components

**108 feature-gated modules** are dead-code-eliminated from the published npm package (internal Anthropic modules for daemon, proactive notifications, context collapse, coordinator mode, etc.). These cannot be recovered.

Build stubs exist in:
- `stubs/bun-bundle.ts` — `feature()` stub (always returns `false`)
- `stubs/macros.ts` — MACRO version constants
- `stubs/global.d.ts` — Global type declarations

## Analysis Documentation

Detailed architecture and feature analysis in `docs/`:
- `docs/en/` — English reports (telemetry, hidden features, undercover mode, remote control, roadmap)
- `docs/zh/` — Chinese translations
