# Subagent Lifecycle Specification

## 1. Executive Summary

A **Subagent** represents an active execution instance of a `Skill`. This document defines the formal state machine, creation rules, context isolation, pause/checkpoint mechanics, failure recovery, and destruction requirements for Subagents across Android and Desktop JVM runtimes.

## 2. State Machine Specification

Subagents progress through strict, verifiable state transitions:

```
  ┌──────────────┐
  │  UNINITIALIZED│
  └──────┬───────┘
         │ create()
         ▼
  ┌──────────────┐
  │   CREATED    │
  └──────┬───────┘
         │ initialize()
         ▼
  ┌──────────────┐      pause()      ┌──────────────┐
  │    ACTIVE    ├──────────────────>│    PAUSED    │
  │ (EXECUTING)  │<──────────────────┤(CHECKPOINTED)│
  └──────┬───────┘      resume()     └──────┬───────┘
         │                                  │
         │ finish() / cancel() / fail()     │ evict()
         ▼                                  ▼
  ┌──────────────┐                   ┌──────────────┐
  │  TERMINATED  ├──────────────────>│  DESTROYED   │
  └──────────────┘    destroy()      └──────────────┘
```

### State Matrix
| From State | To State | Trigger | Conditions & Handlers |
|---|---|---|---|
| `UNINITIALIZED` | `CREATED` | `SubagentFactory.create()` | Allocates instance, assigns `SubagentId`, pins Skill version. |
| `CREATED` | `ACTIVE` | `initialize()` | Binds `SubagentContext`, verifies capability grants. |
| `ACTIVE` | `PAUSED` | `pause()` | Invokes `onPause()`, persists `CheckpointData` via transaction. |
| `PAUSED` | `ACTIVE` | `resume()` | Re-hydrates state from checkpoint; re-acquires context handles. |
| `PAUSED` | `DESTROYED` | `evict()` | Evicts instance from RAM after verified checkpoint save. |
| `ACTIVE` | `TERMINATED` | `finish()` / `fail()` / `cancel()` | Execution completes, throws exception, or receives cancellation. |
| `TERMINATED` | `DESTROYED` | `destroy()` | Clears `SubagentContext`, detaches ClassLoader, releases memory. |

## 3. Subagent Context & Memory Scope

Each Subagent receives an isolated `SubagentContext` instance providing:
- **`SubagentId`**: System-wide unique identifier (e.g. `subagent_calc_8f9a2b`).
- **`CorrelationId`**: Unique execution correlation token for event propagation.
- **`CapabilityAdapter`**: Restricted handle for requesting actions; does not expose underlying platform accessibility or OS handles.
- **`Logger`**: Scoped event logger appending to the persistent event log.
- **`KeyValueStore`**: Private, sandboxed persistent key-value namespace for skill runtime state.

**Isolation Rule**: Subagents MUST NOT receive direct references to:
1. Other active Subagent instances.
2. The `MainAgent` private state or coroutine scope.
3. Raw Android `Context`, `AccessibilityService`, or Desktop OS process handles.

## 4. Pause, Checkpointing, and Recovery

### 4.1 Cooperative Checkpointing Protocol
When `ResourceManager` signals memory pressure or process death imminent:
1. `SubagentRuntime` calls `subagent.onPause()`.
2. Skill returns a `CheckpointData` object containing key-value pairs representing execution sequence step, pending parameters, and transient calculation variables.
3. `SubagentRuntime` writes `CheckpointData` to persistent storage (`skill_runs` table) within an atomic transaction.
4. Once database commit completes, Subagent transitions to `PAUSED` and may be evicted from memory.

### 4.2 Recovery & Reconciliation
Upon restart or resumption:
1. `SubagentRuntime` reads latest `CheckpointData` from database for `correlationId`.
2. Instantiates fresh Subagent and invokes `onResume(checkpoint)`.
3. **Reconciliation Check**: Before repeating an action, the skill MUST query `SkillExecutor` or target environment to verify if the previous action was already committed (e.g., checking if button click succeeded) to prevent duplicate execution of non-idempotent actions.

## 5. Cancellation, Timeouts, and Destruction

### 5.1 Timeout Enforcement
- Default max task execution timeout is defined in `RESOURCE_BUDGET.md` (default: 60s for UI skills, 300s for batch skills).
- Exceeding execution timeout triggers automatic `cancel(Reason.TIMEOUT)` followed by `destroy()`.

### 5.2 Destruction Cleanup
When a Subagent enters `DESTROYED`:
1. `SubagentContext` is invalidated; further action requests throw `ContextInvalidatedException`.
2. References to loaded classes and resources are detached to permit Garbage Collection.
3. Active threads or coroutines associated with `SubagentId` are forcefully cancelled.
