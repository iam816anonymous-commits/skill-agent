# Security Model Specification

## 1. Executive Summary

SkillAgent implements a multi-layered security model centered on capability-based access control, runtime user consent, mandatory API mediation, dynamic package verification, and immutable audit logging.

## 2. Threat Model & Risk Analysis

| Threat Vectors | Risk Description | Primary Countermeasures |
|---|---|---|
| **Privilege Escalation** | A skill attempts to perform unauthorized operations (e.g. accessing contacts or storage without permission). | Dual-stage capability checks (`Manifest Check` + `Runtime Check`). Protected actions MUST route through `SkillExecutor`. |
| **Bypassing Action Engine** | A skill attempts to directly invoke Android Accessibility APIs or Desktop Robot APIs. | API restriction in `IsolatedClassLoader` plus pure domain core boundaries. Raw OS handles are never passed to skills. |
| **Malicious Skill Package Import** | Importing corrupted or tampered `.skill` packages containing path traversal or unauthorized code. | Zip-slip / path traversal checks during package extraction, SHA-256 hash validation, and explicit user consent prompts showing declared capabilities. |
| **Resource Exhaustion Attack** | A skill allocates excessive memory or spawns infinite loops to freeze the device. | strict memory heap tracking, execution timeouts, cooperative eviction, and thread/coroutine termination. |
| **Cross-Skill Data Leakage** | A skill attempts to inspect or mutate memory or persistent data belonging to another skill. | SubagentContext isolation, database table namespacing, and prohibition of direct peer-to-peer skill messaging. |

## 3. Capability Architecture & Taxonomy

Capabilities are categorized fine-grained permission units:

```
                  ┌─────────────────────────────────────┐
                  │          Capability Categories      │
                  └──────────────────┬──────────────────┘
                                     │
     ┌──────────────────┬────────────┼────────────┬──────────────────┐
     ▼                  ▼            ▼            ▼                  ▼
┌──────────┐      ┌──────────────┐ ┌──────────┐ ┌─────────┐ ┌─────────────────┐
│ HARDWARE │      │ACCESSIBILITY │ │ BROWSER  │ │ STORAGE │ │ SYSTEM_SETTINGS │
└──────────┘      └──────────────┘ └──────────┘ └─────────┘ └─────────────────┘
```

### Capability Authorization Matrix
| Capability | Target Platform | Protection Level | Runtime Grant Mechanism |
|---|---|---|---|
| `HARDWARE_CAMERA` | Android / Desktop | HIGH | Android Runtime Permission Prompt / OS Hardware Grant |
| `ACCESSIBILITY_UI_AUTOMATION` | Android | CRITICAL | Android Accessibility Service Enablement Prompt |
| `BROWSER_NAVIGATE` | Android / Desktop | MEDIUM | Skill Manifest Declaration + Permission Center UI |
| `STORAGE_READ_WRITE` | Android / Desktop | MEDIUM | Android Storage Access Framework (SAF) / OS File Picker |
| `SYSTEM_SETTINGS_MODIFY` | Android / Desktop | HIGH | Android `WRITE_SETTINGS` intent grant / OS Admin |

## 4. Five-Stage Action Authorization Protocol

Before ANY protected action (e.g. `CLICK`, `TYPE`, `LAUNCH_APP`, `BROWSER_NAVIGATE`) is executed by `SkillExecutor`, `CapabilityManager` executes a mandatory five-stage check:

```
 [Action Request]
        │
        ▼
 1. Is capability declared in SkillManifest? ──────────> NO ──> REJECT (Undeclared)
        │ YES
        ▼
 2. Is capability supported on current platform? ─────> NO ──> REJECT (Unsupported)
        │ YES
        ▼
 3. Is system OS permission granted by user? ─────────> NO ──> REJECT (OS Permission Denied)
        │ YES
        ▼
 4. Is underlying service bound & active? ────────────> NO ──> REJECT (Service Unavailable)
        │ YES
        ▼
 5. Is specific action payload authorized? ───────────> NO ──> REJECT (Action Rejected)
        │ YES
        ▼
 [EXECUTE ACTION VIA SKILL EXECUTOR]
```

If any check fails, `CapabilityManager` throws a `SecurityException`, logs the violation to `EventLogger`, and returns a structured error to the caller.

## 5. Runtime User Consent, Revocation & Audit Logging

### 5.1 Permission Center UI
- SkillAgent features a dedicated Permission Center interface displaying installed skills, declared capabilities, current authorization status, and toggle controls.
- **Dynamic Revocation**: Revoking a permission in the Permission Center takes effect immediately. Active Subagents holding the revoked capability receive an `onPermissionRevoked()` callback and are gracefully halted.

### 5.2 Immutable Security Audit Log
Every permission decision (grants, denials, runtime checks, revocations) is recorded in `skill_events` table in SQLite storage with:
- `timestamp`: System time in epoch milliseconds.
- `subagentId`: Calling subagent identifier.
- `capability`: Capability checked.
- `decision`: Enum (`GRANTED`, `DENIED_UNDECLARED`, `DENIED_REVOKED`, `DENIED_OS_PERMISSION`).
- `correlationId`: Workflow trace ID.

## 6. Import Package Security Rules

Dynamic `.skill` package imports MUST satisfy four security invariants:
1. **No Path Traversal**: Zip entries containing `../` or leading root paths (`/etc/`, `C:\`) are rejected immediately.
2. **Hash Integrity Verification**: SHA-256 hash of the imported archive MUST match signature records accompanying the package.
3. **Explicit User Installation Prompt**: Upon import, the UI MUST display the skill's author, ID, description, and complete list of declared capabilities before writing the package to local storage.
4. **Integrity Check != Trust Decision**: Manifest hashes verify package integrity, NOT trustworthiness. Imported executable code remains subject to sandbox isolation and capability checks.
