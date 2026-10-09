# SkillAgent — Jules Engineering Governance & Guidelines

## Mission

Build SkillAgent as a modular, skill-based agent runtime for Android API 27–36+ and Desktop JVM (JDK 21). Skills are manifest-defined capabilities executed through a controlled runtime, universal action engine, permission manager, orchestrator, persistent storage, and resource manager.

The authoritative roadmap and all phase-specific specifications are located in `docs/`.

## Mandatory Phase Governance

Before creating, editing, deleting, moving, or generating any project file:

1. Read `docs/PHASE_SCOPE_GOVERNANCE_RULE.md`.
2. Identify the currently authorized phase.
3. Read the specifications and acceptance criteria applicable to that phase.
4. Inspect the repository's current state and existing implementation.
5. Produce a **Phase Alignment Check** before any file changes.

The **Phase Alignment Check** must list:
- Current phase and status.
- Authorized inputs and dependencies.
- Files permitted to change.
- Required deliverables.
- Explicitly prohibited future-phase work.
- Authorized tests.
- Exit criteria and required evidence.
- Repository conflicts, missing prerequisites, and blockers.

If the governance document is missing, contradictory, or insufficient to determine the authorized scope, stop and report the issue. Do not silently create a replacement rule or proceed based on assumptions.

## Scope Enforcement

Implement only the active phase.

- Do not pull forward future-phase functionality, even when convenient.
- Do not introduce speculative abstractions, unapproved dependencies, unrelated refactors, or broad rewrites.
- Read access does not imply edit authorization. Test authorization does not imply permission to change tests.
- If an earlier-phase defect blocks the current phase, report it and request authorization before modifying out-of-scope files.
- **Do not advance automatically to the next phase.** Wait for explicit authorization after completing a phase gate.

## Engineering Standards

- Follow the frozen specifications in `docs/` and established repository conventions.
- Use pure Kotlin/JVM for the domain core. Core contracts must be free of `android.*` classes, Room, or UI dependencies.
- Route all protected skill actions through `SkillExecutor` and capability checks.
- Treat ClassLoaders as loading mechanisms, not security sandboxes.
- Validate all skill manifests, messages, imported packages, and external inputs.
- Persist recoverable state consistently and avoid replaying non-idempotent actions without reconciliation.
- Keep platform-specific implementations behind approved adapter interfaces.
- Never weaken assertions, remove tests, suppress failures, or fabricate evidence.

## Verification & Completion

- Run tests permitted and authorized by the active phase.
- Audit the final diff for scope compliance, test completeness, and security regressions.
- Record actual evidence in `docs/PHASE_<N>_AUDIT.md` (or the authorized phase audit file).
- Report phase completion with PASS, FAIL, or BLOCKED status and stop.
