# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process that connects to WhatsApp, routes messages to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/whatsapp.ts` | WhatsApp connection, auth, send/receive |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `src/group-queue.ts` | Per-group concurrency, container lifecycle |
| `src/container-runtime.ts` | Runtime abstraction (Apple Container / Docker) |
| `container/skills/*/SKILL.md` | Agent skills (browser automation, uber-eats, etc.) |

## Skills

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
npm run test         # Run all tests (vitest)
npm run test:watch   # Tests in watch mode
npm run typecheck    # Type check without emitting
npm run format       # Auto-format with prettier
npm run format:check # Check formatting (CI)
./container/build.sh # Rebuild agent container
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## Architecture

**Message flow:** Baileys (WhatsApp Web) → SQLite → polling loop → GroupQueue → container spawn → Claude Agent SDK → response via IPC

**Container isolation:** Each group gets its own Linux container with 2 GB RAM. Groups see only their own folder (`/workspace/group`), the main group also gets the full project (`/workspace/project`). Skills from `container/skills/` are synced into every container's `.claude/skills/` at startup (`container-runner.ts:140-150`).

**IPC:** Agents write to `data/ipc/{group}/messages/*.json` and `data/ipc/{group}/tasks/*.json` to send messages and schedule tasks. The host watches these directories and processes them.

**State:** SQLite via better-sqlite3. Tables: chats, messages, scheduled_tasks, task_run_logs, router_state, sessions. Group registration lives in `data/registered_groups.json`.

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
