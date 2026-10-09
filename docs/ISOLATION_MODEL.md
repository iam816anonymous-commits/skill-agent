# Isolation Model Specification

## 1. Executive Summary

Executing skill code requires balancing dynamic adaptability with strict system security and process stability. This document defines the trust boundaries, ClassLoader isolation mechanics, process separation models, and Inter-Process Communication (IPC) contracts across SkillAgent targets.

## 2. Trust Boundaries & Skill Categories

SkillAgent establishes three explicit trust levels:

```
  ┌──────────────────────────────────────────────────────────┐
  │ Level 0: MainAgent Core & Trusted Runtime                │
  │ • Full access to internal database, orchestrator, state. │
  └────────────────────────────┬─────────────────────────────┘
                               │
  ┌────────────────────────────┴─────────────────────────────┐
  │ Level 1: Built-in Trusted Skills                         │
  │ • Bundled in app binary. In-memory execution.             │
  │ • Sandboxed via SubagentContext & CapabilityAdapter.    │
  └────────────────────────────┬─────────────────────────────┘
                               │
  ┌────────────────────────────┴─────────────────────────────┐
  │ Level 2: Dynamic Imported Skills (.skill packages)       │
  │ • Loaded dynamically via IsolatedClassLoader / Worker.  │
  │ • Enforced ClassLoader & Capability boundaries.          │
  └──────────────────────────────────────────────────────────┘
```

### Critical Security Rule
**A ClassLoader alone is NOT a security sandbox.** ClassLoader isolation prevents class name collisions, enables dynamic loading/unloading, and restricts direct object references. However, ClassLoader isolation does NOT prevent unauthorized native code execution, CPU exhaustion, memory consumption attacks, or direct reflection attempts. Therefore, dynamic imported code MUST be wrapped with strict runtime capability checks, context sanitization, and where configured, separate process boundaries.

## 3. ClassLoader & Memory Isolation

### 3.1 `IsolatedClassLoader` Design
- Extends JVM/Android `URLClassLoader` or `PathClassLoader`.
- **Parent-Last Delegation Policy for Skill Classes**: Forces the ClassLoader to resolve skill-bundled classes from its local archive first before delegating to the parent ClassLoader.
- **Parent-First Delegation for Core API Contracts**: Directs all core domain interface lookups (`SkillInput`, `SkillOutput`, `CapabilityAdapter`, `SkillExecutor`) to the main application ClassLoader to ensure contract object equivalence across boundaries.
- **Package Blacklist**: Explicitly blocks dynamic classes from loading classes in restricted packages (e.g. `java.lang.reflect.*`, `android.os.Binder`, `dalvik.system.*`, `com.skillagent.core.internal.*`).

### 3.2 Context Invalidation & Memory Reclamation
When a Subagent is destroyed or evicted:
1. All references in `SubagentRuntime` to the Subagent context and instance are set to `null`.
2. `IsolatedClassLoader` is closed (`IsolatedClassLoader.close()`) and unreferenced.
3. Garbage Collection is scheduled to reclaim memory allocated by the uninstalled skill classes.

## 4. Process Isolation & IPC Contracts

### 4.1 Android Process Boundaries
- **In-Process Subagents**: Built-in skills run within the main process using isolated coroutine scopes and `SubagentContext` abstractions.
- **Isolated Service Subagents**: Optional configuration for high-risk dynamic skills using Android's `android:isolatedProcess="true"` service tag. Communication occurs via Android Binder / AIDL IPC with strict permission checks.

### 4.2 Desktop JVM Worker Process Model
On Desktop JVM, imported dynamic skills execute in separate JVM worker processes:
- **Worker Process**: Headless JVM instance started via `ProcessBuilder`.
- **IPC Transport**: Standard Input/Output streams or local TCP sockets bound strictly to `127.0.0.1`.
- **IPC Protocol**: Frame-delimited JSON-RPC messages carrying `correlationId`, `subagentId`, action payloads, and status responses.
- **Process Cleanup**: If the worker process fails to terminate gracefully within 5 seconds of a cancellation signal, `MainAgent` issues `Process.destroyForcibly()`.

## 5. Security & Isolation Invariants

1. **No Shared Mutable State**: Skills MUST NOT share mutable static variables or heap objects with other skills or `MainAgent`.
2. **Context Scope Enforcement**: `SubagentContext` MUST NOT leak references to internal database handles, platform accessibility instances, or OS process controls.
3. **Auditability**: Every attempt to invoke a capability across an isolation boundary MUST be logged via `EventLogger`.
