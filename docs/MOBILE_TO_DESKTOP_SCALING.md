# Mobile to Desktop Scaling Specification

## 1. Executive Summary

SkillAgent is built on a single unified core that runs seamlessly across Android (API 27–36+) and Desktop JVM (JDK 21 Baseline) environments. This specification defines platform-independent contracts, platform-specific runtime behavior, adapter abstractions, and execution scaling rules.

## 2. Platform Architecture & Layering

To guarantee 100% core portability across Android and Desktop, SkillAgent strictly divides system layers into pure core and platform adapters:

```
  ┌──────────────────────────────────────────────────────────┐
  │                 Pure Kotlin Core (100% Portable)         │
  │  SkillDefinition · SkillManifest · GoalState · Subagent  │
  │  Orchestrator · SkillMessageBus · Capability interfaces  │
  └────────────────────────────┬─────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
  ┌───────────────────────────┐   ┌───────────────────────────┐
  │  Android Platform Adapter │   │   Desktop JVM Adapter     │
  │  • Accessibility Service  │   │  • Robot / OS Automation  │
  │  • Android Context / SAF  │   │  • Separate JVM Process   │
  │  • Room / SQLite Adapter  │   │  • SQLite JDBC Adapter    │
  └───────────────────────────┘   └───────────────────────────┘
```

### Pure Core Invariant
The domain core module MUST NOT import `android.*`, `com.google.android.*`, `java.awt.*`, or platform-specific UI frameworks. It relies solely on pure Kotlin standard library and core coroutines.

## 3. Platform Adapter Contracts

All platform-dependent functionality is accessed through platform-independent interfaces:

| Capability Contract | Android Platform Implementation | Desktop JVM Platform Implementation |
|---|---|---|
| **Skill Action Execution** | Translates `CLICK`, `TYPE`, `SCROLL` into Android `AccessibilityNodeInfo` tree actions. | Translates actions into OS native event triggers or browser driver commands. |
| **Capability Authorization** | Queries Android Runtime Permissions + Accessibility Service status. | Queries Desktop Permission Center config + OS Accessibility/Automation grants. |
| **Process / Thread Isolation** | Manages Subagent instances within app process or isolated Android Service. | Spawns separate OS JVM worker processes via `ProcessBuilder` with IPC channels. |
| **Persistent Storage** | SQLite database accessed via Room / Android SQLite driver. | SQLite database accessed via SQLite JDBC driver. |
| **System Event Monitoring** | Receives Android BroadcastReceivers and `onTrimMemory` lifecycle callbacks. | Receives OS signal hooks and JVM ShutdownHooks / MemoryMXBean events. |

## 4. Desktop JVM Worker Process Model

While Android manages subagents within process memory or isolated services under OS lifecycle control, Desktop JVM scales execution using separate worker processes:

### 4.1 Process Isolation Architecture
1. **Worker Process Launcher**: `DesktopSubagentRuntime` uses `ProcessBuilder` to start a headless worker process:
   ```
   java -cp skillagent-worker.jar com.skillagent.desktop.WorkerMain --id <subagent_id> --ipc-port <port>
   ```
2. **IPC Protocol**: Communication between MainAgent and Desktop Worker occurs over local loopback sockets or standard input/output using JSON-RPC or Protobuf frames.
3. **Process Supervision**:
   - MainAgent monitors worker process liveness using `Process.onExit()`.
   - If a worker crashes, MainAgent captures the non-zero exit code, marks the run as interrupted, and initiates `ProcessDeathRecovery` workflow using latest `skill_runs` database checkpoint.

## 5. Mobile-to-Desktop Skill Portability Rules

1. **Write Once, Run Anywhere**: A portable `.skill` package compiled against pure core contracts MUST run unmodified on both Android and Desktop JVM, provided required capabilities exist.
2. **Capability Fallback**: If a skill requests a capability present on Mobile but unavailable on Desktop (e.g. `HARDWARE_TELEPHONY`), `CapabilityManager` immediately rejects the capability check with `UNSUPPORTED_PLATFORM_CAPABILITY`.
3. **No Platform-Specific Code in Skills**: Skills MUST NOT include direct platform checks or calls (e.g. `if (isAndroid) ...`). All platform variations are handled transparently by the host platform adapters.
