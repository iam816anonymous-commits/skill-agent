# SkillAgent

SkillAgent is a universal, modular agent runtime designed to create, run, manage, test, pause, recover, and coordinate isolated skills across Android (API 27–36+) and Desktop JVM (JDK 21) environments.

## Purpose & Target Platforms

SkillAgent provides an agent runtime featuring:
- **Skill System**: Manifest-backed, capability-scoped skill definitions.
- **Permission Model**: Fine-grained, runtime-enforced capability authorization and consent tracking.
- **Shared Action Engine**: Universal skill execution engine routing all platform actions through controlled interfaces.
- **Resource Management**: Dynamic memory tracking, cooperative eviction, state checkpointing, and process-death recovery.
- **Orchestration**: Message-driven workflow engine supporting sequential and parallel skill coordination.
- **Cross-Platform Support**: Portable pure Kotlin core supporting Android API 27–36+ (including low-RAM API 27 Go devices) and Desktop JVM (JDK 21 baseline).

## Architecture Overview

```
MainAgent
 ├── Goal Tracking · Planning · Policy Decisions · Skill Selection
 ├── Orchestrator + Message Bus
 │    ├── Workflow Scheduling · Message Routing · Correlation IDs
 ├── Skill Runtime
 │    ├── Manifest Schema · Subagent Lifecycle · Context · Checkpoints
 ├── Capability Manager
 │    ├── Authorization · Runtime Consent · Revocation · Audit
 ├── Universal SkillExecutor
 │    ├── Target Resolution · Action Validation · Target Re-acquisition
 └── Storage + Resource Manager
      ├── Persistent State · Event Logging · Checkpoint Recovery · Memory Budgets
```

## Phase-Based Development Roadmap

1. **Phase 0** — Specification and Skill Taxonomy
2. **Phase 1** — Foundation and Domain Core
3. **Phase 2** — Persistent Storage and Logging
4. **Phase 3** — Permission and Capability Manager
5. **Phase 4** — Subagent Runtime
6. **Phase 5** — Universal Skill Execution Engine
7. **Phase 6** — Orchestration and Message Bus
8. **Phase 7** — Resource Hardening and Process-Death Recovery
9. **Phase 8** — Desktop and Large-Device Scaling
10. **Phase 9** — Console and Movable Overlay
11. **Phase 10** — Skill Marketplace and Dynamic Loading
12. **Phase 11** — Automated Skill Test Center
13. **Phase 12** — Master Certification: Android to Desktop

All development follows strict governance defined in `docs/PHASE_SCOPE_GOVERNANCE_RULE.md`.
