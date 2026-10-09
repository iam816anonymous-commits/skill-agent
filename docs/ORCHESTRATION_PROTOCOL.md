# Orchestration Protocol Specification

## 1. Executive Summary

The `Orchestrator` and `SkillMessageBus` coordinate execution across multiple Subagents, handling complex multi-stage workflows, parallel task execution, message routing, correlation tracking, and failure handling.

## 2. SkillMessageBus Architecture

The `SkillMessageBus` is a thread-safe, coroutine-backed in-memory and persistent message routing system.

```
                   ┌───────────────────────┐
                   │       MainAgent       │
                   └───────────┬───────────┘
                               │ Workflow Dispatch
                               ▼
 ┌──────────────────────────────────────────────────────────┐
 │                    Orchestrator                          │
 └─────────────────────────────┬────────────────────────────┘
                               │ Enqueue / Route
                               ▼
 ┌──────────────────────────────────────────────────────────┐
 │                  SkillMessageBus                         │
 │   ┌──────────────────┐        ┌──────────────────────┐   │
 │   │ Priority Queue   │        │ Correlation Tracker  │   │
 │   └──────────────────┘        └──────────────────────┘   │
 └───────┬──────────────────────────────────────────┬───────┘
         │                                          │
         ▼ Deliver                                  ▼ Deliver
 ┌──────────────────────┐                  ┌──────────────────────┐
 │ Subagent A (Search)  │                  │ Subagent B (Extract) │
 └──────────────────────┘                  └──────────────────────┘
```

## 3. Message Envelope & Protocol

Every message dispatched on `SkillMessageBus` MUST comply with the standard message envelope:

```kotlin
data class Message(
    val messageId: String,          // UUID v4
    val correlationId: String,      // Workflow tracing tree ID
    val parentMessageId: String?,   // Parent step message ID (for trees)
    val from: String,               // Sender ID (e.g., "main_agent", "subagent_calc_123")
    val to: String,                 // Recipient ID or "orchestrator"
    val type: MessageType,          // Message type enum
    val payload: String,            // JSON-encoded payload
    val timestamp: Long             // System time in epoch millis
)
```

### 3.1 Supported Message Types
- `CREATE`: Instantiates a new Subagent for a specific Skill.
- `EXECUTE_TASK`: Dispatches a task payload to an active Subagent.
- `RESULT`: Subagent returns successful completion output.
- `ERROR`: Subagent signals recoverable or terminal failure.
- `REQUEST_DATA`: Subagent requests workflow data or state from MainAgent.
- `CANCEL`: Forcefully halts task execution for a given `correlationId`.
- `PAUSE_ACK` / `RESUME_ACK`: Checkpointing state transition acknowledgments.

## 4. Message Routing & Mediation Rules

1. **No Direct Peer-to-Peer Skill Communication**: Skills MUST NOT dispatch direct messages to other skills. All data exchange and requests MUST be routed through `Orchestrator` / `MainAgent`.
2. **Correlation ID Propagation**: Every child message spawned during a workflow MUST inherit the root workflow's `correlationId`.
3. **Queue Bounding**: `SkillMessageBus` maintains bounded queues (default capacity: 100 messages per Subagent). Queue overflows trigger backpressure or task suspension.

## 5. Workflow Execution Engine

The `WorkflowEngine` supports sequential and parallel DAG (Directed Acyclic Graph) task execution:

### 5.1 Sequential Workflow Pattern
Example: `SEARCH → EXTRACT → SOLVE`
1. Orchestrator executes `SEARCH` skill with input.
2. Waits for `RESULT` message; validates output schema.
3. Passes validated output as `SkillInput` into `EXTRACT` skill.
4. Passes `EXTRACT` output into `SOLVE` skill.

### 5.2 Parallel Workflow Execution & Join Behavior
1. Orchestrator splits workflow into independent branches.
2. Dispatches task messages concurrently to multiple Subagents.
3. Collects branch results using a barrier join with configured timeout.
4. If a branch fails, Orchestrator evaluates the workflow error policy:
   - `FAIL_FAST`: Cancels all sibling branches immediately.
   - `BEST_EFFORT`: Waits for remaining branches and aggregates available data.

## 6. Failure Handling, Deadlocks, and Timeouts

1. **Deadlock Prevention**: Cyclic workflow dependencies are rejected during workflow DAG validation before execution starts.
2. **Step Timeouts**: If a Subagent fails to respond with `RESULT` or `ERROR` within the step timeout (default: 120s), `SkillMessageBus` automatically synthesizes a `TIMEOUT_ERROR` message and cancels the Subagent.
3. **Duplicate Message Suppression**: Messages received with duplicate `messageId` values within a 1-hour window are dropped automatically.
