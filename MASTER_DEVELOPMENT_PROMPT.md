# HELIOS-X — Master Development Prompt

## Version : 2.0 
**Document:** `MASTER_DEVELOPMENT_PROMPT.md`  
**Purpose:** The first instruction document for a developer or AI coding agent starting work on HELIOS-X.  
**Role:** Define how to inspect the repository, read and reconcile project documents, make safe implementation decisions, maintain project state, and follow the authoritative development and operational references.  
**Operating rule:** Audit → read documents serially → record requirements → reconcile contracts → follow the implementation plan → test → report.  

> This document governs the development process. It does not replace `IMPLEMENTATION_PLAN.md`, `OPERATIONS.md`, or the technical specifications. It deliberately does not repeat their detailed phases, commands, schemas, API contracts, or operational procedures.

---

# PART A — PURPOSE AND WORKING RULES

## 1. Your Role

Act as the lead implementation owner for the actual HELIOS-X repository. Your responsibility is to inspect the existing project, understand its approved specifications, implement the required behavior, run the available checks, and maintain an accurate record of what is complete.

When filesystem and terminal tools are available:

- Inspect existing files before editing.
- Read the required project documents completely and in the order defined in Part B.
- Preserve existing user work, useful code, data, and uncommitted changes.
- Implement real functionality rather than stopping at proposals or generating empty directory structures.
- Run relevant tests and inspect the resulting diffs.
- Report only actions and results that were actually completed or verified.

When filesystem or terminal tools are unavailable, state the limitation and provide a concrete patch or file content. Never claim that files were changed or tests passed when they were not.

## 2. What HELIOS-X Is

HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects — is a scientific platform focused primarily on SPHEREx observations, with NASA/IPAC IRSA as the intended primary archive.

The product has two connected but independently usable capabilities:

1. **Public SPHEREx Sky Image Explorer:** the primary public-facing experience. A visitor enters sky coordinates (RA/Dec), selects a supported field of view, discovers available observations, and inspects actual image frames in observation-time order using the timeline and viewer.
2. **Agentic Scientific Investigation Engine:** a separate, explicitly initiated scientific workflow. A user may start an investigation directly or explicitly hand off a supported Explorer selection. The system validates prerequisites, runs supported deterministic scientific methods, coordinates the eight logical agent roles, preserves evidence and provenance, assesses candidates for human review, and generates traceable reports.

The Public Explorer must remain usable without signing in, creating an investigation, or invoking agents. Browsing, querying, frame navigation, playback, and blinking must not implicitly start an investigation. An explicit, validated handoff or direct investigation request is required.

The three frontend applications have separate responsibilities:

- `frontend/` — Public SPHEREx Sky Image Explorer.
- `user-console/` — authenticated user and investigation workspace.
- `admin-console/` — restricted administrative interface, limited to approved and implemented operational features.

The MVP backend is a single FastAPI modular monolith. The eight agent roles are logical roles defined in `AGENT_SPECIFICATIONS.md`; they are not eight independently deployed microservices. System services such as the Workflow Orchestrator and deterministic Scientific Engine must remain distinct from agent roles as defined by the architecture and agent specifications.

For detailed architecture, product scope, folder ownership, implementation order, persistence design, and operations, use the authoritative reference files listed in Part B. Do not recreate those specifications inside this prompt.

## 3. Non-negotiable Project and Scientific Rules

These rules apply throughout development:

1. Never claim HELIOS-X has discovered or confirmed Planet X.
2. An unmatched catalogue source is not proof of a new astronomical object.
3. SPHEREx's 102 spectral channels are spectral samples, not 102 temporal observation epochs.
4. Temporal playback must use actual distinct observation timestamps. Do not fabricate missing frames or imply that every sky region has observations at an exact six-month interval.
5. Numerical scientific work—including source detection, photometry, coordinate transforms, image alignment, motion, spectra, uncertainty, and statistics—must be implemented as deterministic, tested computation. An LLM must not invent or replace measurements.
6. Agents may plan, call approved tools, and interpret structured outputs. They must not fabricate observations, identifiers, timestamps, measurements, uncertainties, units, archive results, or catalogue matches.
7. Preserve source provenance, product/release information, quality flags, uncertainty, method/software versions, inputs, limitations, alternative hypotheses, and human-review state where applicable.
8. Missing data is not zero. A failed query is not a successful empty query. A successful empty query is not a discovery.
9. Unsupported operations and insufficient prerequisites must use the outcomes defined by the owning workflow/API/scientific specifications. Do not invent substitute status values.
10. `PRIORITIZED_FOR_REVIEW` means eligible for human attention, not scientific confirmation.
11. Failed, unknown, or inconclusive evidence gates must never silently pass.
12. When live archive capability is unverified or unavailable, use clearly labelled fixtures for development and report live integration as `UNVERIFIED` or `BLOCKED` as appropriate.
13. PostgreSQL is the selected application database. Do not reintroduce SQLite as the application database or automatic fallback. Follow `database/DATABASE_SCHEMA.md` for persistence decisions.
14. Keep the MVP architecture simple and modular. Do not add distributed infrastructure or optional services unless `IMPLEMENTATION_PLAN.md` or an approved, documented decision establishes a real requirement.

## 4. Autonomous Execution and Decision Rules

- Make routine, reversible engineering decisions without repeatedly asking the user.
- Ask only when a genuinely blocking decision cannot safely be inferred. Otherwise, record the assumption and proceed.
- Do not silently change approved product scope, public API contracts, database schema ownership, scientific methods, workflow semantics, or security boundaries.
- If a document or dependency is missing, record it honestly and continue work that is independent of that blocker.
- If credentials, network access, datasets, or external tools are unavailable, isolate the blocker. Do not fabricate a live result.
- Do not promise background work or a later result. Complete the work available in the current working session.
- Do not create speculative modules, services, controls, or infrastructure merely to make the repository look complete.

---

# PART B — INITIAL AUDIT AND REQUIRED DOCUMENT REVIEW

## 5. First Action: Read-only Repository Audit

Before changing code, configuration, or documentation, perform a read-only audit:

1. Confirm the current working directory and repository root.
2. List the project-relevant files and source directories.
3. Check Git status and identify modified, staged, and untracked files.
4. Locate the exact reference documents listed in Section 6.
5. Identify the actual canonical path for each reference. In particular, `database/DATABASE_SCHEMA.md` is the canonical database schema document; do not create a second independently maintained schema document elsewhere.
6. Note missing, duplicated, renamed, or conflicting specification files.
7. Do not delete, reset, overwrite, or normalize existing work during this audit.

If the directory is not a Git repository, record that fact; do not initialize or rewrite repository history without an approved reason.

## 6. Mandatory Serial Reading Order

Read each available document completely, one file at a time, in this exact order. Finish recording the review of one document before opening the next. Do not read these files in parallel, and do not use search snippets, file names, remembered summaries, or partial excerpts as a substitute for a complete review.

| Order | Reference file | What it owns / what to capture |
|---:|---|---|
| 01 | `PRD.md` | Product purpose, users, scope, MVP outcomes, non-goals, success criteria, and prohibited claims. |
| 02 | `SYSTEM_ARCHITECTURE.md` | Component boundaries, three frontend responsibilities, backend architecture, data flows, integrations, trust boundaries, and MVP constraints. |
| 03 | `SCIENTIFIC_METHODOLOGY.md` | Scientific methods, assumptions, inputs, units, uncertainties, quality requirements, evidence gates, limitations, and unsupported cases. |
| 04 | `DATA_SOURCE_SPEC.md` | Authoritative data sources, product/release semantics, retrieval routes, metadata, timestamps, quality fields, provenance, and adapter acceptance criteria. |
| 05 | `INVESTIGATION_WORKFLOW.md` | Investigation stages, task dependencies, state transitions, gates, retry/cancellation behavior, partial failures, and report prerequisites. |
| 06 | `AGENT_SPECIFICATIONS.md` | Eight logical agent roles, typed inputs/outputs, approved tools, permissions, boundaries, prohibited behavior, and model-failure handling. |
| 07 | `IMPLEMENTATION_PLAN.md` | **The sole owner of repository build sequence, implementation phases, task breakdown, dependencies, and phase completion criteria.** Capture its canonical layout and next required task; do not reproduce its phase plan here. |
| 08 | `database/DATABASE_SCHEMA.md` | Canonical persistent entities, fields, relationships, constraints, indexes, lifecycle records, provenance, and migration expectations. |
| 09 | `API_SPECIFICATION.md` | Canonical HTTP endpoint methods/paths, request and response schemas, errors, status values, pagination, idempotency, cancellation/retry, and OpenAPI requirements. |
| 10 | `UI_UX_SPECIFICATION.md` | Screens, routes, navigation, interaction behavior, visual system, responsive/accessibility rules, and loading/empty/error/unsupported states. |
| 11 | `TESTING_AND_ACCEPTANCE.md` | Required unit, integration, API, scientific, frontend, security, and end-to-end tests; fixtures; tolerances; and release blockers. |
| 12 | `SECURITY.md` | Secrets, authentication/authorization, public/private boundaries, validation, artifact access, external retrieval, agent permissions, prompt injection, logging, and security acceptance. |
| 13 | `OPERATIONS.md` | **The sole owner of setup, environment configuration, startup, migration commands, test commands, health checks, debugging, backup, restore, and runtime procedures.** Do not copy operational command lists into this prompt. |
| 14 | `BACKEND.md` | Backend module ownership, service/repository boundaries, implementation guidance, and backend-specific definition of done, subject to the API, database, architecture, security, and testing specifications. |
| 15 | `FRONTEND.md` | Frontend application ownership, shared/frontend module rules, API client/types, route implementation guidance, and frontend-specific definition of done, subject to the UI/UX, API, architecture, security, and testing specifications. |

### 6.1 Missing or Renamed Documents

- If a required document is not at the expected path, search the repository for that exact filename.
- If the file exists under a different path, record the actual path and confirm that it is the intended canonical document.
- If it truly does not exist, record `MISSING` in the review register. Do not claim to have read it.
- Do not create a duplicate document simply to satisfy a filename expectation.
- If a missing document owns a decision required by the next implementation task, mark that task blocked until the minimum decision is reconciled. Continue independent work.
- If a reference uses an old path or technology decision, record the discrepancy and resolve it using the authority rules in Section 8 before implementation.

### 6.2 Additional Project Documents

After the 15 required documents have been reviewed in order, inspect other project-relevant Markdown files before implementation:

1. Follow explicit document dependencies or references from the core specifications first.
2. Read the remaining relevant files one at a time in alphabetical filename order.
3. Record their purpose and whether they are authoritative, supporting, historical, or obsolete.
4. Do not let old duplicates, generated notes, or historical documents silently override a current canonical specification.
5. Do not inspect unrelated personal, academic, vendor, build-output, or dependency documentation merely because it is Markdown.

Also inspect relevant source code, configuration, tests, migrations, and existing implementation notes as required by the task. This does not replace the required document review.

---

# PART C — REVIEW REGISTER AND CONTRACT RECONCILIATION

## 7. Maintain `IMPLEMENTATION_STATUS.md`

Create or update `IMPLEMENTATION_STATUS.md` at the repository root. If a suitable implementation tracker already exists, use it rather than creating a competing tracker. Do not create multiple independent status files.

Maintain a **Document Review Register** with one entry for each required reference file:

| Order | File | Status | Main constraints captured | Conflict / missing note |
|---:|---|---|---|---|
| 01 | `PRD.md` | `NOT_STARTED` / `REVIEWED` / `MISSING` / `CONFLICT` | Factual summary | Note |
| 02–15 | Each required reference file | Same statuses | Factual summary | Note |

For each document, before opening the next one:

1. Read the complete available document.
2. Record its purpose in one or two factual sentences.
3. Record the requirements that directly affect implementation.
4. Record its owned contracts or exact technical decisions where relevant.
5. Record dependencies, missing information, and conflicts.
6. Mark it `REVIEWED` only after the full available content has actually been read.
7. Then proceed to the next document in the prescribed order.

A previous-session summary does not qualify as a full review. If the tool cannot access the complete file, record the limitation honestly.

## 8. Resolve Conflicts by Subject Ownership

The reading order determines when a document is reviewed; it does not make an earlier document the authority for every subject.

Use this ownership map when reconciling requirements:

| Subject | Authoritative reference |
|---|---|
| Product scope and user outcomes | `PRD.md` |
| Overall architecture and component boundaries | `SYSTEM_ARCHITECTURE.md` |
| Scientific validity and method requirements | `SCIENTIFIC_METHODOLOGY.md` and verified source/product documentation |
| Archive access and data-product semantics | `DATA_SOURCE_SPEC.md` |
| Investigation lifecycle, task dependencies, and gates | `INVESTIGATION_WORKFLOW.md` |
| Agent roles, tools, and permissions | `AGENT_SPECIFICATIONS.md` |
| Implementation sequence, phases, and repository build plan | `IMPLEMENTATION_PLAN.md` |
| Persistent entities and schema | `database/DATABASE_SCHEMA.md` |
| HTTP/API contracts and canonical API values | `API_SPECIFICATION.md` |
| User experience and visual behavior | `UI_UX_SPECIFICATION.md` |
| Verification and acceptance requirements | `TESTING_AND_ACCEPTANCE.md` |
| Security requirements | `SECURITY.md` |
| Runtime setup and operational procedures | `OPERATIONS.md` |
| Backend implementation guidance | `BACKEND.md`, subordinate to its governing specifications |
| Frontend implementation guidance | `FRONTEND.md`, subordinate to its governing specifications |
| Cross-document review process and development discipline | This file, `MASTER_DEVELOPMENT_PROMPT.md` |

When a conflict is found:

1. Record the exact conflicting statements and their source files in `IMPLEMENTATION_STATUS.md`.
2. Follow the authority for the affected subject when the resolution is clear and does not change approved product scope.
3. Identify every dependent document or implementation area that must be updated.
4. Do not silently maintain two competing contracts.
5. If the conflict materially changes product scope, scientific validity, database schema, public API, workflow semantics, or security boundaries and cannot be resolved from the ownership map, mark the affected work `BLOCKED` and request/record the required decision.
6. Continue independent work that does not depend on the conflict.

### 8.1 Required Reconciliation Record

After the mandatory review sequence, add a concise **Contract Reconciliation** section to `IMPLEMENTATION_STATUS.md`. Record:

- Canonical repository layout and frontend/backend ownership.
- Canonical product capabilities and the Explorer-to-investigation boundary.
- Canonical database schema location and database technology.
- Canonical API contract and lifecycle/status definitions.
- Canonical scientific methods, prerequisites, and evidence gates.
- Canonical data-source/product/release assumptions and verification state.
- Canonical implementation sequence and the next task from `IMPLEMENTATION_PLAN.md`.
- Security and operational prerequisites relevant to the next task.
- Missing/conflicting documents and any decisions still required.
- Commands/tests that remain unverified, referring to `OPERATIONS.md` for how to run them.

Do not reproduce full phase plans, database schemas, API definitions, security checklists, or operational runbooks in the reconciliation record. Link or name their owning documents instead.

## 9. No-Implementation-Before-Reconciliation Rule

Do not begin feature implementation, create database tables, introduce API routes, or build UI features until:

- The read-only repository audit is complete.
- All available required documents have been reviewed serially.
- The review register is current and factual.
- Conflicts affecting the intended first implementation task are resolved or explicitly isolated.
- The canonical repository and database locations are understood.
- The next implementation task and its acceptance criteria are identified from `IMPLEMENTATION_PLAN.md` and the relevant owner specifications.

This rule does not prevent the initial repository audit, document review, status-register maintenance, or work that is demonstrably independent of a documented blocker.

---

# PART D — HOW TO USE THE AUTHORITATIVE DEVELOPMENT REFERENCES

## 10. `IMPLEMENTATION_PLAN.md` Owns the Build Sequence

After document reconciliation, use `IMPLEMENTATION_PLAN.md` as the sole source for:

- Repository structure and directory responsibilities.
- Development phase order and task sequence.
- Dependencies between phases.
- Required deliverables for each phase.
- Phase completion criteria and release gates.
- The next implementation task to execute.

Do not create a second phase plan inside this master prompt, `IMPLEMENTATION_STATUS.md`, or another document. Do not reorder, skip, or invent implementation phases for convenience. If the plan conflicts with an authoritative product, architecture, scientific, database, API, security, or testing contract, record and reconcile the conflict before proceeding.

At the beginning of a development session, identify the current phase and next incomplete task from `IMPLEMENTATION_STATUS.md` and `IMPLEMENTATION_PLAN.md`. Verify the claimed previous completion from code, tests, and Git state before continuing; do not trust a status label without evidence.

## 11. `OPERATIONS.md` Owns Runtime Procedures

Use `OPERATIONS.md` as the sole source for:

- Development prerequisites and environment setup.
- Backend and frontend startup procedures.
- Environment variable names and configuration examples.
- PostgreSQL initialization and Alembic migration commands.
- Test and build commands.
- Health/readiness checks.
- Logging and debugging procedures.
- Data/artifact storage operations.
- Backup and recovery procedures.
- Optional Docker workflows and operational limitations.

Do not copy command lists, environment templates, migration procedures, or backup instructions into this prompt. If implementation changes make an operational instruction inaccurate, update `OPERATIONS.md` and its dependent references rather than maintaining duplicate instructions here.

## 12. Other Owner Documents

Use each technical specification for the subject it owns. In particular:

- Do not derive new API endpoints or response schemas from frontend assumptions; use `API_SPECIFICATION.md`.
- Do not invent tables or fields outside the canonical schema; use `database/DATABASE_SCHEMA.md`.
- Do not derive scientific measurements from LLM text; use the scientific methodology and deterministic implementation.
- Do not invent agent roles or give agents system-service responsibilities; use `AGENT_SPECIFICATIONS.md` and `SYSTEM_ARCHITECTURE.md`.
- Do not treat a UI mockup or placeholder as a working backend capability.
- Do not claim a test passed because it is listed in a specification; run it and record its actual result.

If an implementation requires a genuine contract change, update the owning specification first or in the same controlled change, then update dependent documents and tests. Do not let implementation code silently become the new, undocumented contract.

---

# PART E — IMPLEMENTATION DISCIPLINE

## 13. Follow the Plan Without Duplicating It

For each implementation task, follow the order, dependencies, and completion criteria in `IMPLEMENTATION_PLAN.md`. This master prompt does not define an alternative phase sequence.

Before a task:

1. Re-read the relevant requirement from its owning reference document.
2. Inspect the related existing code, tests, and current Git diff.
3. Confirm the task is the next eligible task under `IMPLEMENTATION_PLAN.md`.
4. Confirm prerequisites and contract decisions are available.

During a task:

- Make the smallest coherent change that satisfies the requirement.
- Follow the canonical repository layout and module ownership.
- Keep frontend, backend, database, scientific computation, orchestration, and agent responsibilities separated as specified.
- Use actual data and persisted resources when available; label fixtures and sample data honestly.
- Keep external provider access behind the approved data adapter boundary.
- Keep scientific computation deterministic and independently testable.
- Keep agent permissions and tools bounded by the agent/security specifications.
- Keep private-resource authorization in the backend, not merely in frontend navigation.
- Avoid unrelated refactoring and unnecessary dependencies.

After a task:

1. Add or update relevant tests.
2. Run the applicable checks using the procedures in `OPERATIONS.md` and the criteria in `TESTING_AND_ACCEPTANCE.md`.
3. Inspect changed files and diffs for unrelated edits, placeholders, secrets, data loss, and contract drift.
4. Record actual results and remaining limitations in `IMPLEMENTATION_STATUS.md`.
5. Mark work complete only when the applicable acceptance criteria are verified.

Do not complete a task by generating files or directories alone. Completion means the required behavior exists and its applicable checks have actually run.

## 14. Maintain `IMPLEMENTATION_STATUS.md`

Keep this file as the single factual progress record. Maintain at least:

- Document Review Register.
- Contract Reconciliation.
- Current implementation phase and source task reference.
- Completed work and changed files.
- Actual verification commands and results.
- Known limitations, blockers, and assumptions.
- Next eligible task, linked to `IMPLEMENTATION_PLAN.md`.

Use only factual labels:

- `PASS` — the command/test was run and passed.
- `FAIL` — the command/test was run and failed.
- `NOT RUN` — not executed.
- `BLOCKED` — cannot execute due to a named blocker.
- `UNVERIFIED` — an external capability/integration has not been demonstrated.
- `PLANNED` — specified but not implemented.
- `REVIEWED` — a document was read in full.
- `MISSING` — a required document was not found.
- `CONFLICT` — an unresolved document or contract conflict exists.

Never report `PLANNED`, `NOT RUN`, `BLOCKED`, or `UNVERIFIED` as complete. Do not create a second tracker for the same progress data.

## 15. Change Safety

Before substantial edits:

- Inspect Git status and the relevant files.
- Understand existing behavior, data structures, and tests.
- Preserve uncommitted work and user data.
- Avoid destructive operations unless necessary and safe.
- Do not reset or rewrite the repository to match a clean template.
- Do not overwrite a canonical specification with a generated duplicate.
- Review the final diff.
- If an API or schema change is required, update the owning specification and dependent contracts consistently.
- Do not make migrations that silently break persisted data.

If a task is blocked by an unavailable credential, network route, data product, tool, or unresolved contract, record the exact blocker and continue independent work where possible.

## 16. Session Handoff and Progress Report

At the end of each substantial working session, update `IMPLEMENTATION_STATUS.md` and provide a factual report containing:

```text
HELIOS-X Development Report

Document review:
- Last required document reviewed:
- Review register state:
- Missing/conflicting references:

Current implementation position:
- Phase/task from IMPLEMENTATION_PLAN.md:
- Gate status:

Implemented in this session:
- Specific behavior and files changed:

Verification:
- Exact commands/tests run:
- PASS:
- FAIL:
- NOT RUN/BLOCKED:

Integration state:
- Public Explorer:
- Investigation workflow:
- PostgreSQL/persistence:
- Fixture vs live archive integration:

Remaining:
- Exact incomplete work and blockers:

Next eligible task:
- One next task from IMPLEMENTATION_PLAN.md:
```

Keep this report short and factual. Do not restate the full implementation plan or operational guide.

When development resumes in a later session, read `IMPLEMENTATION_STATUS.md`, verify the last claimed completion against the repository and tests, and continue from the next eligible task. Do not restart completed work or skip unresolved prerequisites.

---

# PART F — FINAL DEVELOPMENT PRINCIPLE

HELIOS-X must be developed as one coherent, scientifically honest platform with two connected but independently usable capabilities: the Public SPHEREx Sky Image Explorer and the explicitly initiated Agentic Scientific Investigation Engine.

The Public Explorer must not depend on the LLM or agent runtime to browse and display supported observations. The investigation capability must begin only through an explicit, validated request or Explorer handoff. The backend remains a modular monolith for the MVP; the three frontends remain separately buildable; PostgreSQL remains the selected relational database; scientific measurements remain deterministic; and large scientific artifacts remain outside relational rows.

Do not use this prompt as a replacement for the detailed implementation plan or operational instructions. The rules are:

- **This file (`MASTER_DEVELOPMENT_PROMPT.md`):** how the developer/AI starts, reviews documents, reconciles contracts, makes safe changes, and maintains progress.
- **`IMPLEMENTATION_PLAN.md`:** what to build, in what order, and how each implementation phase is accepted.
- **`OPERATIONS.md`:** how to configure, run, test, migrate, debug, back up, and recover the application.
- **The remaining reference documents:** the authoritative product, architecture, scientific, data, workflow, agent, database, API, UI/UX, testing, security, backend, and frontend requirements for their respective subjects.

---

# PART G — STARTING COMMAND TO THE DEVELOPER OR AI CODING AGENT

**Begin now. Follow these instructions in order:**

1. Treat this `MASTER_DEVELOPMENT_PROMPT.md` as the starting instruction, not as permission to begin coding immediately.
2. Perform the read-only repository audit in Section 5. Preserve all existing work.
3. Locate the 15 reference documents in Section 6, including the canonical `database/DATABASE_SCHEMA.md` path.
4. Read each available reference document completely, one at a time, in the exact order listed. After each file, update its entry in `IMPLEMENTATION_STATUS.md` before opening the next file.
5. Record missing or conflicting documents honestly. Do not invent file contents or claim to have reviewed unavailable files.
6. Inspect additional relevant project Markdown files only after completing the required sequence.
7. Complete the contract reconciliation in Section 8 and record the result in `IMPLEMENTATION_STATUS.md`.
8. Identify the first or next eligible implementation task from `IMPLEMENTATION_PLAN.md`. Do not copy or invent a separate phase sequence.
9. Use `OPERATIONS.md` for the actual environment, startup, migration, test, and recovery procedures.
10. Implement the next eligible task, run its required checks, inspect the diff, and update `IMPLEMENTATION_STATUS.md` with factual results.
11. Continue only when the applicable task/phase gate in `IMPLEMENTATION_PLAN.md` and the relevant owner specifications is satisfied, or when a blocker is explicitly recorded and independent work can continue.

**Core instruction:** Read and reconcile before coding; follow the implementation sequence from `IMPLEMENTATION_PLAN.md`; follow runtime procedures from `OPERATIONS.md`; preserve the authority of each reference document; verify actual behavior; and never report unverified work as complete.

**End of Master Development Prompt.**
