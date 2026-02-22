# Bun Port Analysis

Analysis of porting NanoClaw from Node.js to Bun. Written Feb 2026 against the current dependency set and architecture.

---

## Go/No-Go

**Gating factor: Baileys.** Everything else is straightforward or actually an improvement. If `@whiskeysockets/baileys` works on Bun, the port is viable. If it doesn't, there's no reasonable alternative WhatsApp Web client.

**Recommended first step:** Spin up a throwaway Bun project, install Baileys, and test a WhatsApp connection. That one test determines whether any of the rest matters.

---

## Dependency Compatibility

### Production

| Package | Version | Status | Notes |
|---------|---------|--------|-------|
| `@whiskeysockets/baileys` | ^7.0.0-rc.9 | ⚠️ Unknown | Low-level socket/stream handling for WhatsApp Web protocol. No confirmed Bun support. **Primary blocker.** |
| `better-sqlite3` | ^11.8.1 | ❌ → ✅ | Native C++ addon won't work, but Bun has built-in `bun:sqlite` with a nearly identical API. This is actually a win — faster and no native compilation. |
| `cron-parser` | ^5.5.0 | ✅ | Pure JS |
| `pino` | ^9.6.0 | ✅ | Pure JS |
| `pino-pretty` | ^13.0.0 | ✅ | Pure JS |
| `qrcode` | ^1.5.4 | ✅ | Pure JS |
| `qrcode-terminal` | ^0.12.0 | ✅ | Pure JS |
| `yaml` | ^2.8.2 | ✅ | Pure JS |
| `zod` | ^4.3.6 | ✅ | Pure JS |

### Dev

| Package | Status | Bun Equivalent |
|---------|--------|----------------|
| `tsx` | Remove | `bun run` (native TS) |
| `vitest` | Remove or keep | `bun test` built-in, or vitest works too |
| `typescript` | Keep for `tsc --noEmit` | Bun compiles TS natively but doesn't type-check |
| `prettier` | Keep | Works as-is |
| `@types/node` | Remove | Bun has its own types |
| `@types/better-sqlite3` | Remove | Bun's sqlite is typed |

### Node.js Stdlib Usage

All compatible — Bun implements these:

- **child_process** — `spawn()`, `exec()`, `execSync()` used in container-runner, whatsapp, container-runtime
- **fs** — used across 8+ files
- **path** — everywhere
- **os** — `homedir()`, `platform()` in config
- **process** — `cwd()`, `env`, `exit()`, `argv`, signal handlers, `getuid()`/`getgid()`
- **events** — `EventEmitter` in tests
- **readline** — `createInterface()` in whatsapp-auth

ESM with `.js` extensions already in use — works with Bun's module resolution out of the box.

---

## Required Changes

### 1. SQLite — `src/db.ts`

Replace `better-sqlite3` with `bun:sqlite`. The API surface is nearly identical (`.prepare().run()`, `.prepare().get()`, `.prepare().all()`).

```typescript
// Before
import Database from 'better-sqlite3';

// After
import { Database } from 'bun:sqlite';
```

This is the biggest code change but it's well-contained — all SQLite is in `src/db.ts` (~640 lines). Transaction handling and error semantics may differ slightly and need testing.

### 2. Package scripts — `package.json`

```json
{
  "scripts": {
    "dev": "bun run src/index.ts",
    "auth": "bun run src/whatsapp-auth.ts",
    "start": "bun dist/index.js",
    "test": "bun test"
  }
}
```

Remove `tsx`, `@types/node`, `@types/better-sqlite3`, `better-sqlite3` from dependencies.

### 3. TypeScript config — `tsconfig.json`

```json
{
  "module": "ESNext",
  "moduleResolution": "bundler"
}
```

Currently `NodeNext` / `NodeNext` — switching to bundler-style resolution matches Bun's behavior.

### 4. Container image — `container/Dockerfile`

```dockerfile
# Before
FROM node:22-slim

# After
FROM oven/bun:latest
```

The entrypoint changes from `node /tmp/dist/index.js` to `bun /tmp/dist/index.js`.

**Decision point:** The agent-runner inside the container (`container/agent-runner/`) could stay on Node or also move to Bun. Staying on Node is simpler; moving to Bun is more consistent but separate work.

---

## What Doesn't Change

Everything else. The IPC file watcher, message router, group queue, task scheduler, container spawning logic, signal handling — all pure TypeScript/JS using compatible stdlib APIs. No code changes needed.

---

## Migration Phases

### Phase 1: Baileys Smoke Test

Create a minimal Bun project, install Baileys, test WhatsApp connection + QR auth + message send/receive. This is the go/no-go gate.

### Phase 2: Database Layer

Swap `better-sqlite3` → `bun:sqlite` in `src/db.ts`. Test all operations: reads, writes, transactions, schema creation. Run the full test suite.

### Phase 3: Tooling

Update `package.json` scripts, remove Node-specific dev deps, update `tsconfig.json`. Verify `bun test`, `bun run dev`, `tsc --noEmit` all work.

### Phase 4: Container

Update Dockerfile base image. Test container builds, agent spawning, IPC communication. Decide whether agent-runner moves to Bun or stays on Node.

### Phase 5: Integration

End-to-end test: WhatsApp message → SQLite → polling loop → container spawn → Claude agent → IPC response → WhatsApp reply. Verify group isolation, concurrency, scheduled tasks.

---

## Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Baileys incompatible with Bun | **High** | Test first. No workaround if it fails. |
| `bun:sqlite` edge cases differ from `better-sqlite3` | Low | API is similar. Contained to `src/db.ts`. Test thoroughly. |
| Container base image differences | Medium | Test on Linux. May need different system deps for Chromium. |
| Bun ecosystem maturity | Low | Bun is stable for server workloads. Node fallback always available. |

---

## Why Bother

- **Faster startup** — Bun starts in milliseconds vs Node's seconds
- **Native TypeScript** — No `tsx` compilation step, no `tsc` build for dev
- **Built-in SQLite** — Faster than `better-sqlite3`, no native compilation, no `node-gyp`
- **Built-in test runner** — Drop vitest dependency
- **Smaller container images** — `oven/bun` is lighter than `node:22-slim`
- **Lower memory** — Bun typically uses less memory for equivalent workloads
