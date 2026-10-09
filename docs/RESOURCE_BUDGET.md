# Resource Budget and Management Specification

## 1. Executive Summary

SkillAgent is engineered to run on constrained platforms, specifically Android API 27 Go Edition devices with as little as 4 GB RAM (and restricted heap limits per process) up to high-performance Desktop JVM environments. This document defines explicit resource budgets, memory metrics, eviction policies, and measurement contracts.

## 2. Resource Profiles & Acceptance Targets

The following resource budgets are **acceptance targets** that must be measured and verified on target hardware/profiles during Phase 7 and Phase 12 certification.

| Target Environment | Max Active Subagents | Max Heap per Subagent | Max Total Agent Heap | Target PSS / Process Memory |
|---|---|---|---|---|
| **Android Go Low-RAM (API 27)** | 2 Subagents + MainAgent | 35 MB | 150 MB total | < 220 MB total PSS |
| **Android Standard (API 28–36+)** | 3 Subagents + MainAgent | 50 MB | 250 MB total | < 350 MB total PSS |
| **Desktop JVM (JDK 21 Baseline)** | Dynamic (default 8) | 128 MB | 1024 MB total | Scaled to OS RAM |

### Metric Definitions
- **Java / Kotlin Heap**: Runtime memory allocated by JVM/ART garbage collector (`Runtime.getRuntime().totalMemory() - freeMemory()`).
- **PSS (Proportional Set Size)**: Android process memory metric accounting for shared private and proportional dirty native memory.
- **Total Physical Device Memory**: Overall device RAM, checked via Android `ActivityManager.MemoryInfo` or Desktop OS metrics.

## 3. Memory Pressure Levels & Eviction Policy

`ResourceManager` continuously monitors memory metrics and maps system status into three operational levels:

```
    NORMAL (Heap < 70%) ──> WARNING (Heap 70–85%) ──> CRITICAL (Heap > 85% or LowMemory)
```

### 3.1 Memory Pressure Levels & Action Protocol
1. **NORMAL Level**:
   - Subagent creation and task execution proceed without restriction.
2. **WARNING Level**:
   - `ResourceManager` stops admitting new optional background tasks.
   - Triggers explicit Garbage Collection request (`System.gc()`).
   - Warns active Subagents to trim non-essential in-memory caches.
3. **CRITICAL Level** (or OS `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)`):
   - Immediately initiates cooperative eviction protocol.
   - Selects candidate Subagent using LRU (Least Recently Used) modified by Subagent priority.
   - Requests candidate Subagent to pause and serialize `CheckpointData`.
   - Upon confirmed checkpoint database commit, forcefully terminates Subagent instance and unloads ClassLoader.

## 4. Concurrency, Timeouts, and Throttling

### 4.1 Execution Timeouts
- **UI Action Timeout**: 10 seconds per individual `SkillAction` (e.g. waiting for clickable element).
- **Subagent Task Execution Timeout**: 60 seconds default (configurable up to 300 seconds for background batch skills).
- **Orchestration Workflow Step Timeout**: 120 seconds per step.

### 4.2 Concurrency & Dynamic Scaling
- On **Android**, concurrent active Subagent threads are capped strictly by the profile limits in Section 2.
- On **Desktop JVM**, max concurrency dynamically defaults to `min(Available Processors - 1, Total System RAM / 512 MB)`.

## 5. Measurement & Compliance Testing

1. **Automated Stress Testing**: Phase 7 stress tests must execute 1,000 consecutive skill actions and simulated memory pressure triggers without OutOfMemoryError (OOM) crashes.
2. **Measurement Reporting**: Reports MUST measure actual heap and PSS using platform diagnostics (`Debug.getNativeHeapAllocatedSize()`, `MemoryInfo`, `ManagementFactory.getMemoryMXBean()`).
3. **Hardware Honesty Rule**: Benchmark results from desktop simulators or high-end emulators CANNOT be reported as Android Go physical device results. Physical device metrics must be identified explicitly.
