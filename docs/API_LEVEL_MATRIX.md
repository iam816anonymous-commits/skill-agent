# API Level Matrix & Cross-Platform Compatibility Specification

## 1. Executive Summary

SkillAgent runs across a broad span of Android platform versions (API 27 Oreo Go up to API 36+ Android 16+) and Desktop JVM environments (JDK 21 Baseline). This document defines supported platform features, OS restrictions, fallback behavior, and certification criteria across API levels and Desktop targets.

## 2. Platform Feature & Restriction Matrix

| Platform / API Level | Primary Target Profile | Accessibility UI Automation | Security & Storage Model | Process & Background Restrictions | Dynamic Capability Behavior |
|---|---|---|---|---|---|
| **Android API 27 (Oreo 8.1 Go)** | Low-RAM (4 GB RAM / 64 GB storage). Max 35 MB heap per subagent. | Standard Accessibility Node Tree (`AccessibilityNodeInfo`). No gesture APIs. | Legacy Android Permissions (`READ/WRITE_EXTERNAL_STORAGE`). Single `agent.db` SQLite storage. | Strict background service execution limits. Process kill priority under memory pressure. | Dynamic `.skill` classloading via `PathClassLoader`. Basic capability checks. |
| **Android API 28–29 (Pie / Q)** | Mid-tier mobile devices. | Adds basic gesture dispatch (`dispatchGesture`). | Scoped Storage introduced (API 29). `agent.db` Room persistence with WAL mode. | Background activity launch restrictions introduced. | Runtime capability prompts for scoped storage SAF access. |
| **Android API 30–33 (R / S / Tiramisu)** | Standard modern mobile. | Enhanced accessibility node hierarchy and window tracking. | Scoped storage strictly enforced. `<queries>` package visibility restrictions. | Background notification and foreground service type requirements. | Per-app language, runtime permission auto-reset support. |
| **Android API 34–36+ (Upside Down Cake / 15 / 16+)** | High-end modern mobile. | Advanced UI automation and predictive back support. | Granular media permissions, mandatory overlay consent (`SYSTEM_ALERT_WINDOW`). | Foreground service launch restrictions from background. | Dynamic code loading (`DexClassLoader`) strict read-only file permissions enforced. |
| **Desktop JVM (JDK 21 Baseline)** | OS Desktop (Linux, macOS, Windows). | OS native automation / Browser automation (No Android Accessibility APIs). | OS file permissions & sandboxing. `agent.db` accessed via SQLite JDBC driver. | Multi-threaded process isolation. Spawns JVM worker processes via `ProcessBuilder`. | Dynamic concurrency scaling based on system CPU cores and host RAM. |

## 3. Persistent Database Architecture (`agent.db`)

1. **Single Authoritative Database (`agent.db`)**:
   - `agent.db` is the sole authoritative SQLite database supporting both agent state and skill execution.
   - The roadmap proposal of `skill.db` is consolidated into logical tables (`skills`, `subagents`, `skill_runs`, `skill_events`) within `agent.db` to prevent dual-copy state drift.
2. **Platform Persistence Adapters**:
   - **Android**: Accesses `agent.db` via Android Room / SQLite Driver.
   - **Desktop JVM**: Accesses `agent.db` via SQLite JDBC Driver.
3. **Database & WAL Size Quotas**:
   - Total `agent.db` file size plus Write-Ahead Log (WAL) MUST NOT exceed 25 MB on Android Go devices.
   - `LogRetentionManager` automatically prunes `skill_events` older than 7 days or when row count exceeds 10,000 events.

## 4. Feature Fallback & Degradation Strategy

When `CapabilityAdapter` checks a requested capability on a specific platform/API level, it enforces feature fallbacks:

```
                          [Capability Check Requested]
                                       │
                  ┌────────────────────┴────────────────────┐
                  ▼                                         ▼
         [Android Platform]                         [Desktop JVM]
                  │                                         │
    ┌─────────────┴─────────────┐                           │
    ▼                           ▼                           ▼
[API >= 29]               [API 27 Go]               [No Accessibility]
Use Scoped Storage        Use Legacy Permissions    Route via OS / Browser
SAF File Pickers          Direct Path Access        Native Automation
```

- **Gesture Fallback**: On API 27 where `dispatchGesture` is limited, `SkillExecutor` falls back to discrete click node actions (`performAction(ACTION_CLICK)`).
- **Desktop Accessibility Fallback**: On Desktop JVM where Android Accessibility nodes do not exist, `CapabilityAdapter` routes UI actions to OS native automation drivers or browser automation channels.

## 5. Certification Tier Mapping

Cross-platform certification (Phase 12) evaluates targets across four explicit test tiers:
- **Tier A**: Core domain unit tests, manifest schema validation, pure Kotlin logic (Runs on JDK 21).
- **Tier B**: Persistence migrations, IPC protocol validation, message bus routing, permission checks (Runs on Android API 27 emulator + Desktop JDK 21).
- **Tier C**: End-to-end skill execution, UI target resolution, Calculator skill execution (Runs on Android API 27–36+ devices + Desktop OS).
- **Tier D**: Low-memory stress testing, process-death recovery, checkpoint verification (Validated on Physical API 27 Android Go 4 GB RAM device).
