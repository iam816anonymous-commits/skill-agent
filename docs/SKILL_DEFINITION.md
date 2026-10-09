# Skill Definition Specification

## 1. Executive Summary

In SkillAgent, a **Skill** is a modular, manifest-backed, permission-scoped capability unit. A skill is not merely an LLM prompt or script; it is a structured software component that defines typed inputs, expected outputs, declared capabilities, execution constraints, lifecycle hooks, and checkpointing logic.

## 2. Core Concepts & Taxonomy

### 2.1 Skill vs Subagent vs Task
- **Skill**: A static, versioned blueprint containing a manifest (`manifest.json`), declared capabilities, configuration schema, and code/action definitions.
- **Subagent**: An active runtime instance spawned by `MainAgent` to execute a specific Skill under an isolated `SubagentContext`.
- **Task / Run**: A discrete invocation of a Subagent with a specific input payload, bound to a unique `correlationId` and state tracking record in storage.

### 2.2 Skill Classifications
1. **Built-in / Trusted Skills**: Pre-compiled skills packaged directly with the application binary. Execute within the primary process or trusted runtime container with verified capabilities.
2. **Imported / Dynamic Skills**: User-imported `.skill` packages containing manifest, entrypoints, and assets. Loaded dynamically via `SkillLoader` after signature/hash verification and security sandbox wrapping.

## 3. Skill Lifecycle & Lifecycle Hooks

Every Skill must implement or declare compliance with the standardized lifecycle contract:

```
 UNLOADED ──> REGISTERED ──> INITIALIZED ──> EXECUTING ──> PAUSED/CHECKPOINTED
                    │              │              │                 │
                    ▼              ▼              ▼                 ▼
                 DISABLED        FAILED       TERMINATED       DESTROYED
```

### Lifecycle Methods
- `onRegister(context: RegistryContext)`: Invoked when the skill is registered into `SkillRegistry`. Validates compatibility and registers capabilities.
- `onInitialize(context: SubagentContext)`: Prepares runtime environment, allocated buffers, and local state prior to task execution.
- `onExecute(input: SkillInput): SkillOutput`: Main execution entrypoint. All side-effect actions MUST be dispatched through `SkillExecutor`.
- `onPause(): CheckpointData`: Called by `ResourceManager` during memory pressure or process death warning. The skill serializes its internal state into a key-value `CheckpointData` map.
- `onResume(checkpoint: CheckpointData)`: Re-instantiates skill state from serialized checkpoint. Re-validates target state before resuming execution without duplicating non-idempotent actions.
- `onDestroy()`: Releases local handles, clears caches, and invalidates references.

## 4. Inputs, Outputs, and Contracts

### 4.1 Input Schema (`SkillInput`)
- `correlationId`: String (UUID v4) tracing the workflow tree.
- `parameters`: Key-Value map of JSON-serializable primitives, strings, or structured objects adhering to the manifest's `input_schema`.
- `environmentContext`: Immutable platform snapshot (e.g. current screen orientation, display density, platform type).

### 4.2 Output Schema (`SkillOutput`)
- `status`: Enum (`SUCCESS`, `FAILURE`, `CANCELLED`, `RECOVERY_REQUIRED`).
- `resultData`: Structured payload conforming to `output_schema`.
- `error`: Structured error object (`code`, `message`, `isRecoverable`, `stackTraceTraceId`).
- `metrics`: Execution metadata (execution time ms, action count, peak memory).

## 5. Architectural Restrictions & Invariants

1. **No Direct System API Manipulation**: Skills MUST NOT invoke platform UI or accessibility APIs directly (e.g. `AccessibilityNodeInfo.performAction()`, `Desktop Robot API`). All interactions MUST go through `SkillExecutor`.
2. **No Direct State Mutation**: Skills MUST NOT mutate `MainAgent` state or another skill's internal state directly.
3. **Capability Boundary**: Actions requested by a skill MUST belong to the capabilities declared in its `SkillManifest`. Undeclared actions are rejected at execution time by `CapabilityManager`.
4. **Pure Core Portability**: Domain definitions (`SkillDefinition`, `SkillManifest`, `SkillInput`, `SkillOutput`) MUST be pure Kotlin/JVM without `android.*` or third-party UI dependencies.
