# Phase Scope Governance Rule Specification

## 1. Executive Summary

This document defines the strict engineering governance and phase scoping rules for SkillAgent development. To prevent scope creep, architectural drift, accidental refactories, and unverified work, all development activities (including AI agents such as Jules and human contributors) MUST comply with this governance rule.

## 2. Core Governance Rules

### Rule 1: Phase Alignment Check Requirement
Before creating, editing, deleting, moving, or generating ANY project file:
1. Read `docs/PHASE_SCOPE_GOVERNANCE_RULE.md`.
2. Identify the currently authorized phase.
3. Inspect repository state and confirm that all predecessor phase exit gates have been satisfied and certified.
4. Generate a formal **Phase Alignment Check** in the phase report.

The **Phase Alignment Check** MUST contain:
- Active Phase ID and Phase Name.
- Approved Inputs and Prerequisites.
- Permitted Deliverables and Authorized Files.
- Explicitly Prohibited Future-Phase Work.
- Authorized Test Suites.
- Exit Criteria and Required Evidence Files.

### Rule 2: Strict Phase Boundary Enforcement
- Implement ONLY the deliverables authorized for the active phase.
- **NEVER pull forward future-phase code or functionality**, even if doing so appears convenient or trivial.
- Read access to files does NOT grant authorization to edit those files.
- Test execution authorization does NOT grant permission to modify tests or lower assertion thresholds.

### Rule 3: Stop Conditions & Blocker Escalation
Immediately STOP implementation and report a blocker if:
1. Required predecessor phase specifications or exit gates have NOT passed.
2. The repository state contradicts frozen specifications in `docs/`.
3. The assigned task requires modifying files explicitly prohibited in the active phase scope.
4. Missing dependencies or environment constraints prevent running phase-authorized tests.

### Rule 4: No Unilateral Refactoring or Architectural Alterations
- Do NOT refactor unrelated modules, rewrite existing architecture, or delete existing functionality to make a test pass.
- Fix defects ONLY within the authorized active phase scope.

### Rule 5: Honest Verification and Evidence Gate
- All test results MUST be recorded accurately and honestly.
- A failed test CANNOT be reported as a pass.
- A desktop JVM test or simulator test CANNOT be reported as physical Android low-RAM hardware certification.
- Never delete, disable, or suppress failing tests to achieve a passing report status.

### Rule 6: Explicit Authorization for Phase Transition
- **Do NOT advance automatically to the next phase** upon completing an active phase.
- Stop and wait for explicit human user authorization before commencing subsequent phases.

## 3. Detailed Phase Scope & Boundary Matrix

| Phase ID | Phase Name | Primary Scope & Authorized Deliverables | Strictly Prohibited Work | Required Exit Evidence File |
|---|---|---|---|---|
| **Phase 0** | Specification & Governance | Freeze 10 docs in `docs/`, `PHASE_0_AUDIT.md`, minimal `README.md`, `AGENTS.md`. | NO Kotlin source files, NO Gradle scripts, NO runtime modules, NO code generation. | `docs/PHASE_0_AUDIT.md` |
| **Phase 1** | Foundation & Domain Core | Pure Kotlin models (`SkillDefinition`, `SkillRegistry`, `GoalState`, `SubagentId`, `ExecutionLock`). Unit tests. | NO Android APIs, NO Room DB, NO UI, NO accessibility code. | `docs/PHASE_1_AUDIT.md` |
| **Phase 2** | Storage & Persistence | Database schema, SQLite/Room persistence (`skills`, `subagents`, `skill_runs`, `skill_events`), `EventLogger`, migrations. | NO accessibility automation, NO UI overlay, NO dynamic loaders. | `docs/PHASE_2_AUDIT.md` |
| **Phase 3** | Permission Manager | `SkillPermissionManager`, capability adapters, Permission Center scope. | NO dynamic `.skill` imports, NO process workers. | `docs/PHASE_3_AUDIT.md` |
| **Phase 4** | Subagent Runtime | `SubagentRuntime`, `SubagentFactory`, `SubagentContext`, ClassLoader mechanics, lifecycle management. | NO UI overlay, NO marketplace dynamic loader. | `docs/PHASE_4_AUDIT.md` |
| **Phase 5** | Universal Action Engine | `SkillExecutor`, sealed `SkillAction`, `TargetResolver` integration, Calculator action path. | NO multi-agent message bus workflows. | `docs/PHASE_5_AUDIT.md` |
| **Phase 6** | Orchestration & Message Bus | `Orchestrator`, `SkillMessageBus`, `WorkflowEngine`, DAG execution, correlation routing. | NO dynamic memory eviction tuning. | `docs/PHASE_6_AUDIT.md` |
| **Phase 7** | Resource Hardening & Recovery | `ResourceManager`, `RamMonitor`, LRU checkpoint eviction, `ProcessDeathRecovery`. | NO Desktop worker processes, NO marketplace. | `docs/PHASE_7_AUDIT.md` |
| **Phase 8** | Desktop Scaling | `DesktopSubagentRuntime`, Desktop JVM `ProcessBuilder` worker, IPC sockets. | NO UI overlay, NO marketplace import UI. | `docs/PHASE_8_AUDIT.md` |
| **Phase 9** | Console & Overlay | Console CLI commands (`skill list`, `run`, `kill`), movable overlay UI controls. | NO ZIP archive parser, NO marketplace server. | `docs/PHASE_9_AUDIT.md` |
| **Phase 10** | Marketplace & Dynamic Loading | `SkillLoader`, SAF file picker, ZIP manifest validation, SHA-256 package checks. | NO contract test suite discovery engine. | `docs/PHASE_10_AUDIT.md` |
| **Phase 11** | Skill Test Center | `SkillTestRunner`, skill contract test suite execution, failure report generator. | NO master certification report creation. | `docs/PHASE_11_AUDIT.md` |
| **Phase 12** | Master Certification | Tier A–D certification runs on physical Android Go device and Desktop JVM. | NO scope expansion or new feature additions. | `docs/MASTER_CERTIFICATION_REPORT.md` |

## 4. Operational Checklist for Phase Approval

When reviewing a phase completion report, human reviewers verify:
- [ ] Phase Alignment Check reported and verified.
- [ ] No out-of-scope files created or modified.
- [ ] All phase-specific tests pass without assertion weakening.
- [ ] Audit evidence file (`docs/PHASE_<N>_AUDIT.md`) complete with command logs and measurements.
- [ ] Final status explicitly reported as `PASS`, `FAIL`, or `BLOCKED`.
