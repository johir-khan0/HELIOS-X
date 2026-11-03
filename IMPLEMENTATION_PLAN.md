
# HELIOS-X — Implementation Plan
## Version : 2.0
## 1. Project Delivery Strategy

HELIOS-X will be developed as a modular, testable, and auditable platform with two integrated but operationally separate capabilities:

1. **Public SPHEREx Sky Image Explorer — primary public experience and NASA challenge deliverable.** Users enter Right Ascension (RA) and Declination (Dec), select a supported field of view (FOV), discover actual SPHEREx observations for that region, and inspect available image frames in acquisition-time order through a timeline and video-like viewer.
2. **Agentic Scientific Investigation Engine — advanced scientific investigation capability.** Users explicitly promote a selected sky region, observation sequence, or supported target into a bounded investigation. The system validates data and capabilities, runs supported deterministic scientific analyses, coordinates the eight logical AI-agent roles, preserves evidence and provenance, evaluates alternative explanations, prioritizes candidates for human review, and produces traceable reports.

Both capabilities share data-source adapters, observation/product metadata, scientific processing, artifact storage, quality information, and provenance. The Public Explorer must work independently: browsing, querying, frame navigation, playback, and blinking must not create investigations or invoke agents. The investigation path begins only after a direct investigation request or explicit Explorer handoff.

Development must proceed from document reconciliation, repository structure, data feasibility, and contracts to PostgreSQL persistence, a working coordinate-to-observation-to-frame Explorer path, deterministic scientific methods, evidence/provenance, workflow orchestration, agent integration, the authenticated User Console, reporting, and end-to-end validation.

The initial implementation must use a **modular monolith**, not a distributed microservices architecture. The three frontend applications are separately buildable; the backend remains one FastAPI application for the MVP.

### 1.1 Core Technology Stack

HELIOS-X will use a modular technology stack, with each technology assigned a specific responsibility.

#### Frontend Development

- **React** — Build the Public Explorer, authenticated User Console, and restricted Admin Console as separately buildable applications.
- **TypeScript** — Provide type safety across frontend components and application logic.
- **Vite** — Configure frontend development environments and production builds.
- **Tailwind CSS** — Implement consistent UI styling and responsive layouts.

#### Backend Development

- **Python** — Implement application services, workflow orchestration, adapters, and scientific functionality.
- **FastAPI** — Expose backend services through a structured REST API.
- **Pydantic** — Validate request data, application schemas, and structured agent outputs.
- **OpenAPI** — Document API endpoints and request/response contracts through FastAPI.

#### Scientific Computing and Data Processing

- **NumPy** — Handle numerical arrays and scientific computations.
- **SciPy** — Provide applicable numerical methods and statistical functions.
- **Astropy** — Support astronomical coordinates, units, time handling, WCS-related operations, and other relevant astronomy functions where appropriate.

Scientific libraries must be selected according to the chosen data product and implemented analysis requirements. Do not add a library solely because it is listed here.

#### Database and Persistence

- **PostgreSQL** — The selected primary relational database for local development, CI/integration tests, staging, and production.
- **SQLAlchemy 2.x** — Manage typed database models, queries, sessions, and persistence operations.
- **Alembic** — Manage database schema migrations.
- **psycopg** — Use a maintained PostgreSQL driver through the SQLAlchemy PostgreSQL dialect; pin the selected version in the backend dependency configuration.

The database layer must preserve investigation history, Explorer-related metadata/references where persistence is required, scientific evidence, task records, result provenance, and versioned reports. Large FITS files, image arrays, previews, and derived artifacts must remain in controlled file/object storage rather than relational rows.

#### Agent Integration

- **Provider-independent LLM interface** — Allow agent implementations to use a replaceable model provider without coupling scientific logic to a specific provider.
- **Structured outputs** — Ensure agent responses follow validated schemas and reference supporting evidence.
- **Eight logical agent roles** — Implement the roles defined in `AGENT_SPECIFICATIONS.md`; they do not require eight separate LLM processes or microservices.

LLMs will support planning, coordination, interpretation, and evidence-based recommendations. Scientific measurements and statistical calculations will remain in deterministic computational functions. The Workflow Orchestrator, Scientific Engine, Candidate Assessment Service, and Scientific Report Builder are system components/services, not additional AI agents.

#### Testing and Quality Assurance

- **pytest** — Test backend logic, scientific functions, data validation, adapters, and workflow execution.
- **Frontend testing tools** — Validate UI behavior and application components across the three frontend applications.
- **API integration tests** — Verify communication between frontend-facing endpoints, backend services, and PostgreSQL persistence.
- **Scientific acceptance tests** — Validate methods against known-answer fixtures and actual supported data where available.

Testing must cover normal execution, invalid inputs, missing data, empty archive results, failure handling, access boundaries, and scientific validation.

#### Development and Version Control

- **Git** — Track source-code changes and preserve development history.
- **GitHub Actions or equivalent CI** — Automate relevant tests, linting, formatting checks, PostgreSQL migration tests, and builds.
- **Environment configuration** — Keep credentials and environment-specific settings outside source code, using `.env.example` to document required variables.

#### Observability and Deployment

- **Python logging** — Record application events, warnings, errors, and investigation execution details.
- **OpenTelemetry** — Add instrumentation and tracing when operational requirements justify it.
- **Sentry** — Track application errors when appropriate for the deployment.
- **Docker** — Provide reproducible application environments when needed.

### 1.2 Technology Selection Principle

The implementation will prioritize simplicity, reproducibility, and scientific reliability. PostgreSQL is the final selected relational database for HELIOS-X; it is not a future scalability option. Use PostgreSQL consistently in local development, CI/integration tests, staging, and production. Do not configure SQLite as the application database or as an automatic fallback.

The modular monolith is the initial application architecture. Redis, distributed task queues, object storage, or additional infrastructure may be introduced only when a demonstrated workload, deployment, durability, or operational requirement justifies them. PostgreSQL remains the selected relational database if those infrastructure components are added.

No technology should be added solely to make the architecture appear more advanced. Every dependency must support a defined application, operational, or scientific requirement.

Dependencies must be selected according to the chosen data product and actual implementation needs. Not every scientific library or optional infrastructure component must be installed during the first phase.

### 1.3 Governing Development Principles

- Establish the repository structure before implementing application features.
- Before coding, follow `MASTER_DEVELOPMENT_PROMPT.md` to review the project documents in the required serial order and record missing files or unresolved conflicts.
- Treat the Public Explorer as the primary NASA-facing deliverable; do not postpone its coordinate-to-frame path until after the agent platform.
- Keep Explorer browsing independent of investigation creation, LLM availability, and agent execution.
- Define data contracts before parallel development of frontend, backend, and agents.
- Verify SPHEREx product/archive feasibility before implementing live-data or scientific claims.
- Use actual observation/acquisition timestamps for temporal sequences.
- Keep spectral channels, detector arrays, observation epochs, retrieval time, and release/processing version as separate dimensions.
- Keep scientific calculations in deterministic, versioned computational functions.
- Use AI agents for planning, coordination, interpretation, and evidence-based recommendations.
- Keep agent-generated interpretations separate from raw measurements.
- Preserve data provenance, uncertainty, quality outcomes, and historical result versions.
- Implement one working Public Explorer path and one bounded, supported investigation workflow before expanding optional features.
- Use explicit statuses for unsupported analyses, insufficient data, failed tasks, and successful empty archive results.
- Do not claim Planet X or another new planet has been discovered through automated candidate prioritization.
- Avoid premature microservices, distributed queues, Kubernetes, and unnecessary infrastructure.
- Do not create non-functional UI controls or claim features are implemented merely because they appear in a specification.

---

## 2. Phase 0 — Create the Complete Repository and Folder Structure

**This is the first coding task.**

Before starting this task, read `MASTER_DEVELOPMENT_PROMPT.md` and complete its required document-review sequence. Documentation review and conflict reconciliation are prerequisites; repository setup remains the first coding task.

Before implementing scientific functions, agent logic, database operations, or UI features, create the repository skeleton described below.

The purpose is to establish clear boundaries between the three frontend applications, backend, scientific engine, agent roles, database assets, API contracts, documentation, tests, and deployment configuration.

Do not implement every component at this stage. Create the root files and essential directories, initialize project environments, and add README files only where they help explain the structure. Avoid creating large numbers of empty modules merely to make the repository look complete.

### 2.1 Required Repository Structure

Use the following structure as the canonical initial target. Some directories can be added when their corresponding implementation phase begins, but their ownership and intended location must be established from the beginning.

```text
helios-x/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CHANGELOG.md
├── pyproject.toml
├── package.json
├── pnpm-workspace.yaml
├── .gitignore
├── .editorconfig
├── .env.example
├── Makefile
├── docker-compose.yml                 # optional local composition
│
├── frontend/                          # Public SPHEREx Sky Image Explorer
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── app/
│       ├── routes/
│       ├── features/
│       │   ├── sky-search/
│       │   ├── sky-viewer/
│       │   ├── observation-timeline/
│       │   ├── frame-comparison/
│       │   ├── frame-metadata/
│       │   └── capabilities/
│       ├── components/
│       ├── services/
│       ├── hooks/
│       ├── types/
│       ├── utils/
│       ├── styles/
│       └── main.tsx
│
├── user-console/                      # Authenticated researcher/user workspace
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── app/
│       ├── routes/
│       ├── features/
│       │   ├── investigations/
│       │   ├── execution-monitor/
│       │   ├── evidence/
│       │   ├── candidates/
│       │   ├── reports/
│       │   ├── human-review/
│       │   └── user-settings/
│       ├── components/
│       ├── services/
│       ├── hooks/
│       ├── types/
│       ├── utils/
│       ├── styles/
│       └── main.tsx
│
├── admin-console/                     # Restricted operational/admin application
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── app/
│       ├── routes/
│       ├── features/
│       │   ├── system-health/
│       │   ├── source-registry/
│       │   ├── capability-management/
│       │   ├── job-operations/
│       │   ├── user-access/
│       │   ├── audit-log/
│       │   └── storage-monitoring/
│       ├── components/
│       ├── services/
│       ├── hooks/
│       ├── types/
│       ├── utils/
│       ├── styles/
│       └── main.tsx
│
├── backend/                           # One FastAPI modular-monolith application
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── migrations/
│   │   └── versions/
│   ├── src/
│   │   └── helios_x/
│   │       ├── __init__.py
│   │       ├── main.py
│   │       ├── api/
│   │       │   ├── dependencies.py
│   │       │   ├── errors.py
│   │       │   └── routes/
│   │       ├── core/
│   │       │   ├── config.py
│   │       │   ├── logging.py
│   │       │   ├── security.py
│   │       │   └── exceptions.py
│   │       ├── schemas/
│   │       ├── domain/
│   │       ├── database/
│   │       │   ├── base.py
│   │       │   ├── session.py
│   │       │   ├── models/
│   │       │   └── repositories/
│   │       ├── services/
│   │       │   ├── explorer/
│   │       │   ├── observations/
│   │       │   ├── image_artifacts/
│   │       │   ├── investigations/
│   │       │   ├── executions/
│   │       │   ├── evidence/
│   │       │   ├── candidates/
│   │       │   ├── reporting/
│   │       │   └── reviews/
│   │       ├── orchestration/
│   │       ├── science/
│   │       ├── data_sources/
│   │       ├── storage/
│   │       ├── evidence/
│   │       ├── reporting/
│   │       ├── integrations/
│   │       └── observability/
│   └── tests/
│       ├── unit/
│       ├── integration/
│       └── fixtures/
│
├── agents/                            # Eight logical AI-agent roles
│   └── src/
│       └── helios_x_agents/
│           ├── __init__.py
│           ├── base.py
│           ├── registry.py
│           ├── contracts/
│           ├── prompts/
│           ├── providers/
│           ├── tools/
│           └── implementations/
│               ├── investigation_planner/
│               ├── data_retrieval/
│               ├── data_quality/
│               ├── change_detection/
│               ├── motion_analysis/
│               ├── spectral_photometry/
│               ├── catalogue_matching/
│               └── validation_hypothesis/
│
├── database/                          # Canonical database documentation/assets
│   ├── DATABASE_SCHEMA.md
│   ├── seeds/
│   └── sql/
│
├── shared/
│   ├── contracts/
│   │   ├── investigation.schema.json
│   │   ├── task.schema.json
│   │   ├── observation.schema.json
│   │   ├── frame-manifest.schema.json
│   │   ├── evidence.schema.json
│   │   ├── candidate.schema.json
│   │   └── report.schema.json
│   └── fixtures/
│
├── tests/
│   ├── fixtures/
│   │   ├── valid/
│   │   ├── invalid/
│   │   └── synthetic/
│   ├── unit/
│   ├── integration/
│   ├── scientific/
│   ├── api/
│   ├── e2e/
│   ├── acceptance/
│   └── regression/
│
├── docs/
│   ├── PRD.md
│   ├── SYSTEM_ARCHITECTURE.md
│   ├── AGENT_SPECIFICATIONS.md
│   ├── SCIENTIFIC_METHODOLOGY.md
│   ├── INVESTIGATION_WORKFLOW.md
│   ├── IMPLEMENTATION_PLAN.md
│   ├── DATA_SOURCE_SPEC.md
│   ├── API_SPECIFICATION.md
│   ├── TESTING_AND_ACCEPTANCE.md
│   ├── UI_UX_SPECIFICATION.md
│   ├── SECURITY.md
│   ├── OPERATIONS.md
│   ├── BACKEND.md
│   ├── FRONTEND.md
│   ├── MASTER_DEVELOPMENT_PROMPT.md
│   └── decisions/
│
├── scripts/
│   ├── setup_dev.py
│   ├── seed_dev_data.py
│   └── validate_environment.py
│
├── infra/
│   ├── docker/
│   ├── ci/
│   └── deployment/
│
├── data/                              # ignored runtime data and artifacts
│   ├── source_products/
│   ├── cutouts/
│   ├── previews/
│   ├── spectra/
│   ├── cache/
│   └── temp/
│
├── artifacts/                         # generated reports/exports; retention controlled
│   └── .gitkeep
│
└── logs/                              # ignored runtime logs
```

This is the planned repository layout, not a requirement to implement every listed file immediately. The actual codebase should contain only modules that have a defined purpose.

Important ownership rules:

- `frontend/` is the public Sky Image Explorer.
- `user-console/` is the authenticated research workspace.
- `admin-console/` is restricted and is not a NASA P0 deliverable.
- `backend/` is one FastAPI modular-monolith application.
- `agents/` is the single canonical agent package. Do not duplicate agent source under `backend/src/helios_x/agents/`.
- Root `database/DATABASE_SCHEMA.md` is the canonical database specification. Runtime SQLAlchemy sessions, models, and repositories stay under `backend/src/helios_x/database/`; Alembic migrations stay under `backend/migrations/`.
- Do not create or maintain a second independently editable `docs/DATABASE_SCHEMA.md`.
- `shared/contracts/` contains stable cross-boundary contracts. API request/response authority remains `API_SPECIFICATION.md` and implemented OpenAPI.
- `data/`, `artifacts/`, and `logs/` are controlled runtime locations and must be ignored by Git except for approved placeholders/examples.
- Do not create empty placeholder modules for every possible feature.

### 2.2 Directory Responsibilities

#### Root directory

Contains repository-wide configuration, development commands, workspace setup, environment examples, and high-level documentation.

The root `pyproject.toml` may define shared Python tooling, linting, formatting, and test configuration. `backend/pyproject.toml` defines backend package dependencies. Each frontend application's `package.json` defines its own app dependencies. Avoid duplicating dependency definitions unnecessarily.

#### `frontend/`

Contains the complete Public SPHEREx Sky Image Explorer.

- `app/`: application initialization, routing, and providers.
- `features/`: coordinate search, sky viewer, timeline, frame comparison, metadata, and capabilities.
- `services/`: backend API client and endpoint calls.
- `types/`: frontend types aligned with the API/shared contracts.
- `hooks/`: reusable React hooks.
- `styles/`: global styles and design tokens.

The Public Explorer must not perform authoritative scientific calculations or require an investigation record to display images.

#### `user-console/`

Contains the authenticated researcher/user workspace.

- `features/investigations/`: investigation creation and management.
- `features/execution-monitor/`: execution/task status.
- `features/evidence/`: evidence and measurement inspection.
- `features/candidates/`: candidate assessments.
- `features/reports/`: report viewing.
- `features/human-review/`: authorized review workflows.
- `features/user-settings/`: implemented user preferences.

This console uses the same backend API and shared data foundation; it must not duplicate archive retrieval or scientific logic.

#### `admin-console/`

Contains the restricted operational/admin interface. Implement only approved, real operations such as system health, source/capability visibility, job operations, audit records, access management, and storage monitoring. Do not add non-functional controls merely to fill the console. Every enabled administrative action requires backend authorization and audit behavior.

#### `backend/`

Contains the single FastAPI application.

- `api/`: HTTP endpoints, dependencies, request/response handling, and error mapping.
- `core/`: configuration, logging, security, and shared exceptions.
- `schemas/`: Pydantic request, response, and internal data schemas.
- `domain/`: core domain rules and entities.
- `database/`: PostgreSQL sessions, persistence models, and repositories.
- `services/`: Explorer, observation, image-artifact, investigation, evidence, candidate, report, and review services.
- `orchestration/`: task planning, dependencies, execution states, gates, and scheduling logic.
- `data_sources/`: data-source interfaces, capabilities, and adapters.
- `science/`: deterministic scientific calculations and validation functions.
- `storage/`: controlled source-product and derived-artifact storage.
- `evidence/`: evidence ledger, provenance, and result versioning.
- `reporting/`: report assembly and export logic.
- `integrations/`: external integrations that do not belong to a more specific module.
- `observability/`: operational instrumentation.

The backend owns application state and coordinates scientific execution. The API layer must not contain scientific algorithms directly.

#### `agents/`

Contains the eight logical agent roles, their prompts, schemas, provider adapters, and approved tool interfaces. Agent implementations may be invoked in-process through typed interfaces in the MVP. They are not eight mandatory microservices.

#### `database/`

Contains the canonical `DATABASE_SCHEMA.md` and only approved database-level assets. The runtime persistence implementation and migrations remain under `backend/`.

#### `shared/`

Contains stable, machine-readable contracts used across application boundaries, including investigation, task, observation, frame-manifest, evidence, candidate, and report schemas. Do not maintain conflicting copies of the same contract across frontend, backend, and agent code. Prefer generating frontend API types from OpenAPI where practical.

#### `tests/`

Contains cross-module tests:

- `unit/`: individual functions and classes.
- `integration/`: interactions between components.
- `scientific/`: numerical correctness and scientific validation.
- `api/`: HTTP contract and error behavior.
- `e2e/`: cross-application workflows.
- `acceptance/`: full user-facing requirements.
- `regression/`: previously validated behavior.
- `fixtures/`: controlled datasets and synthetic test inputs.

Backend-specific tests may also live in `backend/tests/`. Keep test ownership clear and avoid duplicating the same test in both locations.

#### `docs/`

Contains product, architecture, scientific, API, agent, UI, testing, security, implementation, and operational documentation. The canonical database schema is maintained at `database/DATABASE_SCHEMA.md`, not as a second copy under `docs/`.

The project documents must be reviewed in the order required by `MASTER_DEVELOPMENT_PROMPT.md`. The remaining specifications must be reviewed before their dependent implementation phase begins. Do not treat backend or frontend guides as substitutes for API, security, testing, database, or scientific specifications.

#### `scripts/`

Contains repeatable developer and maintenance scripts. Scripts must be safe to rerun where practical and must not silently overwrite scientific evidence.

#### `infra/`

Contains deployment and infrastructure configuration. Begin with local development and CI. Add production deployment configuration only when there is a concrete deployment target.

#### `data/`, `artifacts/`, and `logs/`

Contain controlled runtime data, generated derivatives/reports, and logs. They must not contain secrets or unapproved committed astronomical data. Retention and provenance follow `OPERATIONS.md`.

### 2.3 Initial Folder-Setup Tasks

Complete these tasks before implementing product functionality:

1. Create the Git repository and top-level directory structure.
2. Initialize the Python backend package.
3. Initialize the three separately buildable React, TypeScript, and Vite applications: Public Explorer, User Console, and Admin Console.
4. Create the root `agents/` package boundary without implementing all agent logic.
5. Create the root `database/` documentation/assets directory and retain `database/DATABASE_SCHEMA.md` as its canonical schema location.
6. Configure environment variables using `.env.example`; do not commit credentials.
7. Add `.gitignore` rules for virtual environments, dependency folders, PostgreSQL dumps, secrets, caches, source products, and generated artifacts.
8. Add basic README instructions for local setup.
9. Configure Python and frontend formatting, linting, and test commands.
10. Add a minimal backend health endpoint and minimal application shells for the three frontend applications.
11. Add CI checks for installation, linting, formatting, and basic tests. PostgreSQL migration tests are added with the database phase.
12. Verify that a clean checkout can reproduce the development setup.

**Completion criteria:** The repository structure exists, the backend and three frontend applications can start independently, configuration is externalized, basic CI checks run, and no scientific functionality is falsely represented as implemented.

---

## 3. Phase 1 — Scope and Data Feasibility

After the repository skeleton exists, determine what the NASA-facing Explorer and the first bounded scientific investigation can actually demonstrate with verified SPHEREx data.

### Work

- Review the NASA SPHEREx challenge statement and its Resources tab; record the exact required and optional objectives.
- Define the core Public Explorer journey: RA/Dec input, supported FOV, actual observation discovery, image-frame sequence, and time-based visual inspection.
- Identify the candidate SPHEREx data products, archive access route, official product documentation, release information, file structure, units, WCS, observation-time metadata, channel/wavelength metadata, quality flags, and calibration/uncertainty fields.
- Verify access and applicable usage restrictions for the selected archive/products.
- Inspect whether the selected products can support coordinate-based region queries and WCS-aware cutouts or display derivatives.
- Determine actual observation coverage for the five NASA test targets: Barnard's Star, Epsilon Indi B, WISE 0855, RW Aurigae, and R Draconis.
- Use the challenge-supplied WISE 0855 coordinate string: RA `08h55m10.831s`, Dec `-7d14m42.53s`.
- Resolve coordinates for targets not specified by the challenge from an authoritative source, recording the source, coordinate frame/reference epoch, and proper-motion treatment where relevant.
- Treat SPHEREx's 102 near-infrared spectral channels as wavelength samples, not 102 observation epochs. Keep observation time, spectral channel/wavelength, the six detector arrays, retrieval time, and product-release/processing version as distinct dimensions.
- Do not assume that the approximately six-month all-sky mapping cadence guarantees multiple usable observations for any requested coordinate.
- Select a documented sample dataset or create clearly labelled synthetic fixtures for deterministic development tests.
- Define one precise scientific question for the first agentic investigation, based on the data capabilities actually available.
- Specify the intended investigation inputs, outputs, and acceptance criteria.
- Define explicit non-goals and known limitations, including unsupported analyses.
- Record assumptions that remain unresolved.

### Required decisions

The coordinate-to-observation-to-image-frame Explorer path is the first NASA-facing implementation target.

Select the first end-to-end investigation based on actual data capabilities, not on the desired feature list. Do not assume the first investigation must include motion analysis, spectral interpretation, or Planet X candidate assessment. A target named by NASA is a validation target, not a guarantee of data coverage or a positive detection.

### Completion criteria

- The NASA challenge requirements are mapped to product requirements and implementation phases.
- The initial SPHEREx product, release, archive route, and official documentation are recorded or explicitly marked unverified.
- The coordinate/FOV query and image-frame feasibility are documented.
- Actual availability outcomes for the five NASA targets are recorded where live archive verification is possible; unavailable coverage remains explicit.
- Required input fields and metadata are listed.
- The first supported Explorer fixture/query scenario is defined.
- The first investigation workflow and its expected outputs are defined.
- Data limitations and unsupported analyses are explicit.
- A sample retrieval or local fixture can be validated.
- Proposed MVP claims are consistent with available evidence.

---

## 4. Phase 2 — Project Foundation and Shared Contracts

### Work

- Finalize configuration and environment handling.
- Establish logging and consistent exception handling.
- Define typed models for sky-region queries, coordinates, FOV geometry, observations, products, image artifacts, frames, frame manifests, investigation source snapshots, investigations, measurements, tasks, evidence, candidates, and reports as required by the approved contracts.
- Define shared status enums and error categories.
- Establish identifier, coordinate, unit, and timestamp conventions.
- Define evidence references, artifact references, and provenance fields.
- Define the immutable Explorer-selection snapshot contract used by an explicit investigation handoff.
- Define task, data-source retrieval, measurement, quality-check, and evidence-gate contracts.
- Define API and agent input/output contracts.
- Generate or validate TypeScript types from the API contract where practical.
- Keep API schemas separate from PostgreSQL persistence models.

### Required contracts

At minimum, establish contracts for:

- Sky-region/Explorer query requests and results.
- Normalized coordinates, FOV geometry, coordinate frame, and time conventions.
- Observation and product references.
- Frame and versioned frame-manifest records.
- Image artifact and controlled delivery references.
- Explorer-to-investigation source snapshots.
- Investigation requests and records.
- Data-source retrieval results.
- Scientific measurements.
- Task requests and results.
- Evidence records and provenance links.
- Candidate assessments.
- Report outputs.
- Quality-check outcomes.
- Evidence-gate outcomes.

Keep task execution statuses distinct from quality-check outcomes, evidence-gate outcomes, Explorer query outcome codes, capability availability, and candidate statuses.

Use the canonical task statuses defined in `SYSTEM_ARCHITECTURE.md`:

`QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRYING`, `CANCELLED`, `BLOCKED`, `NOT_SUPPORTED`, `INSUFFICIENT_DATA`.

Use the exact request/response and error contracts in `API_SPECIFICATION.md`. Use the exact persistence fields and relationships in `DATABASE_SCHEMA.md`. This phase defines the contracts; PostgreSQL models, sessions, and migrations are implemented in Phase 3.

### Completion criteria

- Application components can load configuration consistently.
- Schema validation rejects malformed inputs.
- Coordinate, time, unit, and FOV conventions are explicit.
- Explorer query, observation, frame-manifest, artifact, and source-snapshot contracts are defined.
- Investigation, task, evidence, and provenance relationships are represented in typed contracts.
- Status transitions and error categories are documented.
- Secrets are excluded from version control.
- Frontend, backend, API, agent, and persistence contracts do not contradict one another.

---

## 5. Phase 3 — Database and Persistence Layer

### Work

- Finalize the canonical `database/DATABASE_SCHEMA.md`.
- Implement PostgreSQL as the single primary relational database for local development, CI/integration tests, staging, and production.
- Use SQLAlchemy 2.x typed declarative models and Alembic migrations.
- Configure a maintained PostgreSQL driver through the SQLAlchemy PostgreSQL dialect (for example, `psycopg`) and pin the selected version in the backend dependency configuration.
- Read the database connection URL and credentials from environment/secret configuration; document only placeholders in `.env.example`.
- Implement centralized PostgreSQL session management and repository interfaces.
- Add database migrations and migration tests against clean temporary PostgreSQL databases.
- Implement persistence for the entities defined by `DATABASE_SCHEMA.md`, including investigations, executions/tasks, retrievals, observations/products, measurements, evidence, candidates, reports, and human review records as required by the MVP.
- Add Explorer query/manifest/artifact/snapshot persistence only where required by the canonical schema and reproducibility requirements.
- Establish foreign-key relationships, unique constraints, appropriate indexes, and transaction boundaries.
- Implement result versioning and immutable historical records where required.
- Add data-integrity and duplicate-prevention/idempotency rules.
- Keep large FITS files, image arrays, previews, spectra, and report binaries outside relational rows; persist metadata, IDs, checksums when computed, and controlled artifact references.

### Persistence principles

- PostgreSQL is the canonical relational database. SQLite is not the application database or an automatic fallback.
- Use the same database platform in local development, CI/integration tests, staging, and production to avoid engine-specific behavior differences.
- Scientific result history must not be silently overwritten.
- Evidence must reference its relevant inputs and derived results.
- Re-running an analysis must create a distinguishable run or result version.
- Database persistence must not be coupled directly to LLM provider logic.
- Keep ORM models separate from Pydantic/API contracts.
- Keep transactions short; do not hold database transactions open during archive calls, artifact transfers, image processing, or LLM calls.
- Redis is optional and is not the durable system of record. Do not introduce it without a measured or explicitly required cache, rate-limit, temporary-state, or queue-coordination use case.
- Do not store provider credentials or secrets in database records.
- Database schema ownership remains with `DATABASE_SCHEMA.md`; do not create competing table or field definitions in this plan.

### Completion criteria

- Required entities can be persisted and retrieved through PostgreSQL.
- Relationships and integrity constraints are tested.
- Alembic migrations can be applied to a clean PostgreSQL database.
- Migration tests run against PostgreSQL, not an alternate in-memory relational engine.
- Historical analysis results remain available after re-analysis.
- Duplicate task execution does not unintentionally create inconsistent records.
- Large scientific artifacts are stored outside relational rows and have controlled references.
- Database configuration and backup expectations are documented without exposing credentials.

---

## 6. Phase 4 — Data Adapter and Quality Validation

### Work

- Implement a fixture adapter first, using clearly labelled and versioned test fixtures.
- Implement the documented IRSA/SPHEREx adapter only after verifying the current official archive access route and product documentation.
- Define a common data-source adapter interface.
- Add data-product and operation capability declarations.
- Store retrieval metadata, exact query parameters, source references, product/release identifiers, and retrieval outcomes.
- Validate schemas, units, observation timestamps/time conventions, coordinate systems, WCS, spectral-channel/wavelength metadata, detector-array metadata where supplied, calibration information, and quality flags.
- Inspect actual product structure and file integrity before using a retrieved product.
- Implement missing-data, partial-failure, provider-timeout, corrupt-product, and unsupported-product behavior.
- Distinguish a successful empty observation query from a failed archive request.
- Add integrity checks and checksums where feasible.
- Apply the release-selection and product-compatibility policy defined by `DATA_SOURCE_SPEC.md`; do not silently mix incompatible releases or assume a release/product is available.
- Preserve unfamiliar or unavailable source metadata as unknown/unavailable rather than inventing default values.
- Verify whether the product supports coordinate cutouts and what WCS-aware processing is valid.
- Record actual archive coverage for the five NASA validation targets when live access is available.

### Completion criteria

- Valid input passes the documented checks.
- Invalid input returns a useful failure reason.
- Missing data are not silently replaced.
- Unsupported products, releases, channels, or operations are reported explicitly.
- Retrieval provenance is stored.
- A successful query with zero observations is distinct from a provider failure.
- Data validation outcomes distinguish passed, failed, inconclusive, unavailable, and not-applicable checks.
- Actual observation time is kept separate from retrieval time, release time, channel/wavelength, and detector-array identity.
- Data-source behavior is covered by unit and integration tests.
- Fixture data are never presented as live SPHEREx observations.

---

## 7. Phase 5 — Public SPHEREx Sky Image Explorer

This phase delivers the core NASA-facing product experience. It must be completed independently of the agentic investigation engine. The Explorer does not require an investigation record, login, LLM provider, or agent execution to search and display available public observations.

### Work

#### 7.1 Explorer backend path

- Implement the bounded sky-region query service using validated RA, Dec, coordinate frame, FOV geometry/units, and supported filters.
- Discover actual SPHEREx observations/products through the configured adapter.
- Return truthful zero-, one-, or multiple-observation outcomes.
- Implement the Explorer query status and result outcome behavior defined in `API_SPECIFICATION.md`.
- Build versioned frame manifests from actual returned observations and products.
- Order frames by validated observation/acquisition timestamps, not filename, retrieval time, release time, or spectral channel number.
- Group compatible frames into sequences; do not combine incompatible channel, array, release, calibration, or product dimensions into one misleading timeline.
- Generate or retrieve WCS-aware cutouts and browser-displayable previews when the product supports them.
- Preserve the original scientific product separately from every generated cutout, preview, or comparison derivative.
- Store large products and image artifacts in controlled artifact storage; expose only stable artifact references or authorized delivery endpoints.
- Preserve image dimensions, pixel scale, units, WCS, channel/wavelength, detector-array identity, quality flags, calibration, contamination, variance, PSF, release, and processing metadata where supplied.
- Enforce the configured FOV/area, time-range, frame-count, file-size, query-duration, concurrency, and public rate limits.
- Expose clear loading, partial, empty, unsupported, insufficient-data, invalid-product, and provider-failure states.
- Do not create an investigation, candidate, report, or agent task when a visitor simply searches, opens, plays, steps through, or blinks images.

#### 7.2 Public Explorer frontend

- Initialize `frontend/` as the Public SPHEREx Sky Image Explorer, not as the authenticated investigation dashboard.
- Implement RA/Dec input with supported decimal/sexagesimal formats, clear units, and validation.
- Implement capability-driven FOV selection and only those optional filters advertised by the backend.
- Show query state and actual observation coverage.
- Implement frame selection, actual-time timeline ordering, play/pause, previous/next navigation, and active observation timestamp.
- Implement frame metadata inspection, including product/release, channel/wavelength, detector array where available, WCS and quality/contamination information where supplied.
- Provide blink/rapid-toggle comparison only as a visual inspection aid; it must not imply that the images are astrometrically registered or prove physical motion.
- Do not generate intermediate frames, synthetic timestamps, or regular six-month sequences to fill coverage gaps.
- Support shareable coordinate URLs only when the API/UI contracts define the supported parameters. Do not place secrets, private investigation IDs, or signed artifact URLs in a share URL.
- Keep the Explorer usable when the LLM provider or investigation engine is unavailable.

#### 7.3 Explorer-to-investigation boundary

- Expose an explicit “Investigate” handoff only when the corresponding investigation API and source-snapshot validation are available.
- A handoff must reference the server-validated query, manifest version, selected frames, observations, and products.
- Investigation creation and workflow execution remain separate explicit actions.
- The backend must revalidate all selected references; it must not trust browser-supplied measurements or transient frontend state.
- Ordinary Explorer viewing must never launch agents implicitly.

#### 7.4 NASA validation coverage

- Create fixture-backed tests for Barnard's Star, Epsilon Indi B, WISE 0855, RW Aurigae, and R Draconis.
- Use the challenge-supplied WISE 0855 coordinates exactly as recorded in the PRD.
- Record actual live archive availability separately from fixture-only results.
- Do not require a positive detection or multiple usable epochs for a target when the archive does not provide them.

### Completion criteria

- A visitor can enter a supported RA/Dec position and FOV and submit a bounded query.
- The backend returns actual observation/product records or an honest zero/one/multiple/failed/unsupported outcome.
- A valid frame manifest is returned, versioned, grouped into compatible sequences, and ordered by actual observation time.
- At least one supported fixture-derived image frame can be delivered through controlled artifact access.
- The Public Explorer can display frames and navigate them without an investigation record or agent call.
- The active frame's real observation time and available product metadata are visible.
- Zero-frame and single-frame states do not create synthetic playback.
- A visual blink is not described as a measured motion result.
- Fixture-only results are labelled clearly; live support is claimed only after real archive/product verification.
- The Explorer path has automated API, artifact, UI, and end-to-end tests.

---

## 8. Phase 6 — Scientific Engine Baseline

### Work

With the independent Public Explorer path established, implement only the deterministic scientific functions required by the selected MVP investigation. Do not make basic image browsing or playback depend on these scientific methods.

Potential modules include:

- Coordinate conversion and angular separation.

- Image registration and registration-quality evaluation.

- Source detection and measurement.

- Change detection.

- Motion estimation and uncertainty propagation.

- Photometry, SNR, and variability tests.

- Spectral processing and model fitting.

- Catalogue cross-matching.

- Statistical testing.

- Alternative-hypothesis checks.

- Candidate assessment rules.

These are potential capabilities, not mandatory MVP deliverables. Implementation must follow the selected data product and scientific question. Basic Explorer playback, frame navigation, and blinking are not scientific-engine tasks and must already work independently. Combining data from the six detector arrays, on-demand spectra, image addition/subtraction/stacking, and Gaia/WISE overlays remain conditional optional enhancements; verify product semantics, method validity, and tests before implementing them, and do not let them delay the P0 Explorer.

### Required implementation rules

Every scientific function must define:

- Purpose and supported inputs.

- Input and output schemas.

- Units and coordinate conventions.

- Algorithm and implementation version.

- Assumptions and limitations.

- Uncertainty treatment.

- Validation criteria and numerical tolerances.

- Failure conditions and explicit status behavior.

Scientific functions must not require an LLM to calculate their numerical results.

### Scientific validation tests

Use synthetic inputs with known expected outcomes and documented reference examples where available.

Test:

- Normal inputs.

- Invalid inputs.

- Missing metadata.

- Boundary cases.

- Known coordinate shifts.

- Known flux changes.

- Uncertainty propagation.

- Known catalogue associations and no-match cases.

- Unsupported analysis requests.

- Reproducibility within defined tolerances.

### Completion criteria

- Every implemented function has documented inputs, units, outputs, and failure modes.

- Tests cover normal, edge, and invalid inputs.

- Results include uncertainty or explicitly state why it is unavailable.

- Numerical acceptance criteria are documented before validation results are accepted.

- No LLM is required to execute the scientific calculations.

---

## 9. Phase 7 — Evidence Store and Reproducibility

### Work

- Implement the evidence ledger.

- Store observations, product/frame/artifact references where required, measurements, quality checks, statistical results, and interpretations as appropriately typed records.

- Record input evidence IDs and parent-result relationships.

- Record code and dependency versions.

- Store method parameters and configuration versions.

- Record warnings, errors, and task status.

- Add run versioning and checksums where feasible.

- Make evidence retrievable by investigation, task, and candidate.

- Preserve the distinction between measurements and agent-generated interpretations.

### Completion criteria

- Each reported quantitative value can be traced to its source and method.

- Each derived result identifies its relevant inputs.

- Re-running an analysis preserves the previous result.

- Evidence can be retrieved by investigation and candidate ID.

- Missing provenance is explicitly identified rather than fabricated.

- A report can be assembled from persisted evidence without relying on agent memory alone.

---

## 10. Phase 8 — Workflow Orchestration

### Work

- Implement the Workflow Orchestrator.

- Implement task dependencies and the investigation execution graph for explicit investigations only; Public Explorer queries and playback remain outside the agentic investigation graph.

- Validate plans before execution.

- Implement task state transitions.

- Add timeout, retry, cancellation, and failure-handling policies.

- Add idempotency and duplicate-execution safeguards.

- Implement capability gates and configurable scientific evidence gates.

- Add partial-run and resume behavior.

- Add dependency-aware re-analysis.

- Enforce resource limits and termination conditions.

- For Explorer-origin investigations, consume the immutable, backend-validated source snapshot and revalidate selected observations/products before execution.

### Responsibilities

The Investigation Planner Agent proposes a plan. The Workflow Orchestrator validates and executes it. This workflow is for explicit investigation requests and must not be invoked by normal Public Explorer browsing or playback.

The orchestrator must determine:

- Which tasks are eligible to run.

- Which tasks depend on earlier results.

- Which failures block downstream tasks.

- Which independent tasks may continue.

- When an investigation needs more data.

- When a report can be generated.

- When an investigation must terminate.

### Completion criteria

- Tasks execute in the correct dependency order.

- A failed mandatory step blocks dependent steps.

- Independent tasks can continue when their prerequisites remain valid.

- Retries do not unintentionally duplicate results.

- Unknown gate outcomes remain unknown.

- Cancellation and resource limits behave predictably.

- Re-analysis preserves historical results.

---

## 11. Phase 9 — Agent Integration

Implement the eight specialized agent roles incrementally after their corresponding system interfaces and scientific outputs are stable. Each role is a logical responsibility and does not require its own LLM process, microservice, or deployment. Keep their implementations in the root `agents/` package.

### 9.1 Specialized Agent Roles

1. **Investigation Planner Agent** — Converts the user request into a structured investigation plan.

2. **Data Retrieval Agent** — Coordinates supported data retrieval and interprets retrieval outcomes.

3. **Data Quality Agent** — Reviews validation results and identifies missing or unsuitable inputs.

4. **Change Detection Agent** — Interprets validated source-detection and change-measurement outputs.

5. **Motion Analysis Agent** — Interprets validated motion estimates and their limitations.

6. **Spectral & Photometry Agent** — Interprets validated spectral, flux, and variability results.

7. **Catalogue Matching Agent** — Interprets catalogue associations, ambiguities, and query outcomes.

8. **Validation & Hypothesis Agent** — Evaluates alternative explanations, contradictions, and unresolved evidence.

The Candidate Assessment Service and Scientific Report Builder remain deterministic/application services, not additional AI agents. Candidate scoring must remain deterministic and policy-driven.

The Workflow Orchestrator, Scientific Engine, Data-Source Adapters, Candidate Assessment Service, and Scientific Report Builder are system components/services, not additional AI agents.

### 9.2 Agent Implementation Rules

- Define a common agent interface and registry.

- Use typed, versioned inputs and outputs.

- Restrict agents to approved tools and evidence.

- Validate all structured outputs.

- Record model and prompt versions.

- Apply bounded retries and timeouts.

- Prevent agents from overwriting raw measurements.

- Require evidence references for factual claims.

- Keep agent interpretations separate from scientific results.

- Preserve safe behavior when an LLM provider is unavailable.

- Do not let agents bypass mandatory quality or evidence gates.

Agents may recommend additional analysis, but the orchestrator must validate the recommendation before scheduling execution.

### 9.3 Integration Order

Start with the Planner, Retrieval, Data Quality, and Validation roles. Add the Change Detection, Motion Analysis, Spectral & Photometry, and Catalogue Matching roles when their corresponding scientific interfaces are available.

Do not create artificial LLM calls for tasks that can be completed reliably by deterministic code.

### Completion criteria

- Agent inputs and outputs conform to documented schemas.

- Invalid outputs are rejected or retried within a bounded policy.

- Every factual claim references supporting evidence.

- Agent outputs cannot overwrite scientific measurements.

- A missing LLM response does not corrupt stored scientific results.

- The workflow remains valid when optional agent roles are unavailable.

---

## 12. Phase 10 — Backend API

### Work

Implement the remaining investigation, execution, task, evidence, candidate, report, review, and operational API routes around stable application services and documented contracts. The P0 Explorer query, observation, frame-manifest, and artifact API path is delivered with Phase 5.

Use `BACKEND.md` as the detailed backend implementation guide, while treating `API_SPECIFICATION.md`, `SYSTEM_ARCHITECTURE.md`, `DATABASE_SCHEMA.md`, `SECURITY.md`, and `TESTING_AND_ACCEPTANCE.md` as the governing contracts for their respective subjects. Resolve any contradiction before coding; do not silently invent a new contract.

Implement endpoint groups exactly as defined in `API_SPECIFICATION.md`. Investigation-related groups include:

- Investigation creation and retrieval.

- Investigation execution, cancellation, and status.

- Task status and execution details.

- Observation and measurement retrieval.

- Evidence retrieval.

- Candidate assessment and candidate details.

- Report retrieval and export.

- Human review submission.

- Health and readiness checks.

The P0 Explorer query, observation, frame-manifest, frame metadata, and controlled artifact routes are implemented in Phase 5 according to `API_SPECIFICATION.md`. Do not invent alternative paths or payloads. This phase completes the remaining investigation/execution/task/evidence/candidate/report/review API surface and ensures the implemented OpenAPI contract matches the specification.

Finalize `API_SPECIFICATION.md` before frontend integration becomes extensive.

### API rules

- Validate incoming requests.

- Return explicit error responses and appropriate HTTP status codes.

- Separate API schemas from database persistence models where necessary.

- Enforce backend access control for protected investigations, executions, evidence, candidates, reports, reviews, and administrative resources; apply documented rate/resource limits to public Explorer routes.

- Do not expose secrets or internal credentials.

- Do not perform expensive scientific calculations inside request-handling code when a managed task execution is more appropriate.

- Keep API responses consistent with the canonical task, evidence, and candidate status models.

### Completion criteria

- API endpoints match the documented contracts.

- Request validation and error behavior are tested.

- Scientific functions remain separate from HTTP handlers.

- Investigation status can be queried throughout execution.

- Evidence and report references are available through documented endpoints.

- API documentation is generated and verified.

---

## 13. Phase 11 — User Console, Admin Console, and Frontend Integration

### Work

The Public Explorer is implemented in Phase 5 and remains the primary public entry point. In this phase, complete the authenticated User Console and the separately owned Admin Console boundary using React, TypeScript, Vite, and Tailwind CSS.

Use `FRONTEND.md` as the detailed frontend implementation guide. The Public Explorer is a separate application under `frontend/`; the authenticated investigation workspace is `user-console/`; restricted operations are `admin-console/`. Keep its routes, component boundaries, UI states, and API integration aligned with `UI_UX_SPECIFICATION.md`, `API_SPECIFICATION.md`, and the shared backend contracts. If they conflict, record the discrepancy and resolve it before implementing the affected feature.

Implement the remaining frontend surfaces in the following order:

1. User Console application shell, navigation, and shared layout.

2. Dashboard and investigation list.

3. Investigation creation form.

4. Investigation detail and execution status.

5. Data-quality and evidence views.

6. Measurement and uncertainty views.

7. Candidate assessment view.

8. Report viewer.

9. Human review controls.

10. User settings and configuration views required by the MVP.

Keep `admin-console/` as a separately buildable, restricted application. Implement only approved operational views/actions backed by real protected API behavior. Admin Console functionality is not a NASA P0 deliverable and must not delay the Public Explorer.

### UI requirements

The interface must clearly distinguish:

- Observed data from calculated measurements.

- Measurements from agent interpretations.

- Successful checks from failed or unavailable checks.

- Candidate prioritization from scientific confirmation.

- Completed investigations from incomplete or failed executions.

Display units, uncertainties, data-source references, quality flags, and relevant limitations.

The frontend applications must obtain authoritative scientific results from the backend API. They must not calculate independent scientific results or invent placeholder findings that appear real.

Use clearly labelled synthetic data for development demonstrations.

### Completion criteria

- The Public Explorer remains independently usable and does not depend on User Console authentication.

- An authorized user can create and follow an investigation in the User Console.

- Task and investigation statuses are visible and accurate.

- Measurements and uncertainties are displayed with their units.

- Evidence and report claims can be traced.

- Failed or incomplete tasks are represented honestly.

- Human review actions are recorded by the backend.

---

## 14. Phase 12 — Reporting and Human Review

### Work

- Build reports from persisted evidence.

- Implement report templates and export formats required by the MVP.

- Include data sources, methods, results, uncertainty, and limitations.

- Link quantitative claims to evidence records.

- Record candidate assessment rationale.

- Implement review decisions and annotations.

- Preserve report and review versions.

### Required report content

1. Investigation objective.

2. Data sources and retrieval status.

3. Methods and software versions.

4. Measurements and uncertainties.

5. Catalogue associations and ambiguities.

6. Quality-check outcomes.

7. Alternative hypotheses.

8. Evidence-gate outcomes.

9. Candidate status and prioritization rationale, where applicable.

10. Missing evidence and limitations.

11. Recommended follow-up.

12. Human review status.

### Completion criteria

- Reports are generated from stored data.

- Quantitative claims have evidence references.

- Uncertainty and unavailable checks are visible.

- Report generation does not require an LLM to invent missing information.

- Human review records the decision, rationale, timestamp, and reviewed evidence version.

- Accepting a report as a summary does not automatically confirm its scientific hypothesis.

---

## 15. Phase 13 — Security, Observability, and Operational Readiness

### Work

- Centralize application logging.

- Add request and investigation correlation IDs.

- Record task failures, retry counts, and execution duration.

- Add health and readiness checks.

- Protect environment variables and credentials.

- Validate external input and data-source responses.

- Add dependency and configuration checks.

- Configure CI for tests, linting, formatting, and build verification.

- Add OpenTelemetry and Sentry when their benefits justify the setup.

- Document PostgreSQL backup/restore, migration, recovery, and data-retention expectations for the intended deployment.

- Define public Explorer query/artifact limits separately from protected investigation and administrator access.

- Enforce backend authorization for private investigations, user resources, reviews, and admin operations.

### Completion criteria

- Logs can be associated with the relevant investigation and task.

- Secrets are not committed or exposed in API responses.

- Failures are observable and diagnosable.

- Health checks report meaningful application state.

- CI catches relevant regressions.

- Operational instructions are documented.

---

## 16. Phase 14 — End-to-End Validation and Demonstration

### Work

Run both required product paths: (A) the independent Public Explorer coordinate-to-observation-to-frame workflow and (B) one bounded investigation launched directly or through an explicit validated Explorer handoff. Use live supported SPHEREx products where verified; otherwise use clearly labelled synthetic fixtures.

Test both successful and failure paths for the Explorer and investigation workflows, including:

- Missing or malformed data.

- Unsupported analysis requests.

- Failed data-source queries.

- Successful queries with no results.

- Catalogue-query failure.

- Ambiguous catalogue matches.

- Failed registration or quality checks.

- Insufficient observations.

- Conflicting evidence.

- LLM provider failure.

- Invalid agent output.

- Retry and cancellation behavior.

Also validate the independent Public Explorer flow, frame-manifest ordering, artifact authorization/delivery, zero/one/multiple observation cases, invalid WCS/product behavior, and the explicit source-snapshot handoff.

- Re-analysis after new evidence.

- Report generation and human review.

### Scientific validation

Verify that:

- Known synthetic shifts and flux changes are handled within documented tolerances.

- Uncertainty calculations behave as expected for controlled inputs.

- Statistical functions match trusted reference examples where available.

- Mandatory gates block dependent conclusions when appropriate.

- Results preserve provenance and historical versions.

- Reports do not convert missing evidence into positive evidence.

### NASA target validation

Run and record scenarios for all five NASA challenge targets:

- Barnard's Star.
- Epsilon Indi B.
- WISE 0855, using RA `08h55m10.831s` and Dec `-7d14m42.53s`.
- RW Aurigae.
- R Draconis.

For each target, validate coordinate/name resolution where implemented, query construction, actual archive coverage, frame-manifest ordering, product metadata, and the correct zero/one/multiple/unsupported/insufficient-data behavior. A passing test does not require a positive detection. Do not fabricate observations or assume every target has multiple compatible epochs. Keep fixture results separate from live archive verification.

Also verify that 102 spectral channels are treated as spectral samples, detector arrays remain a separate product dimension, and temporal playback uses only actual observation times.

### Completion criteria

- The full happy path completes.

- Failure paths produce honest, visible statuses.

- The demonstration can be repeated using documented steps.

- The same input, method, and configuration produce results consistent with the defined reproducibility requirements.

- No unsupported scientific discovery claim is made.

- A reviewer can trace each reported measurement to its input and method.

---

## 17. Dependency Order

The recommended implementation sequence is:

```text
Phase 0: Repository Structure and Document Reconciliation
                         ↓
Phase 1: NASA Challenge Scope and SPHEREx Data Feasibility
                         ↓
Phase 2: Shared Contracts and Coordinate/Observation/Frame Schemas
                         ↓
Phase 3: PostgreSQL Persistence and Artifact References
                         ↓
Phase 4: Fixture/IRSA Data Adapters and Metadata Quality Validation
                         ↓
Phase 5: Public Explorer Backend + Coordinate-to-Frame Viewer
                         ↓
Phase 6: Deterministic Scientific Engine
                         ↓
Phase 7: Evidence Store and Reproducibility
                         ↓
Phase 8: Investigation Workflow Orchestration
                         ↓
Phase 9: Eight Logical Agent Roles
                         ↓
Phase 10: Remaining Investigation Backend API
                         ↓
Phase 11: User Console + Restricted Admin Console
                         ↓
Phase 12: Reporting and Human Review
                         ↓
Phase 13: Security and Operational Readiness
                         ↓
Phase 14: Integrated End-to-End and NASA Target Validation
```

Some activities can run in parallel after their interfaces are stable. For example, the three frontend application shells and backend contract work can proceed in parallel after the root structure and shared contracts are agreed. The Public Explorer viewer can be built as soon as the Explorer query, frame-manifest, and artifact contracts are stable; it must not wait for agent integration or advanced scientific analysis.

Do not implement all phases simultaneously. Do not move optional P1 capabilities ahead of the P0 coordinate-to-observation-to-frame path.

---

## 18. Definition of Done for Every Module

A module is complete only when:

- Its purpose and responsibility are documented.

- Its interfaces and data contracts are defined.

- Inputs and outputs are typed and validated.

- Unit tests cover normal and failure cases.

- Errors and warnings are explicit.

- Logs and provenance are adequate.

- Security requirements are satisfied.

- Integration tests pass where applicable.

- The relevant documentation is updated.

- It does not duplicate responsibility already assigned to another module.

For scientific modules, the definition of done additionally requires documented assumptions, units, uncertainty treatment, and validated numerical behavior.

For agent modules, it additionally requires schema validation, bounded execution, evidence-linked claims, and protection against modifying raw scientific results.

---

## 19. Testing Strategy

### 19.1 Unit Tests

Test mathematical functions, coordinate conversion, FOV validation, timestamp normalization, frame ordering, cache keys, uncertainty propagation, schema validation, scoring rules, artifact access rules, and status transitions.

### 19.2 Integration Tests

Test data adapters, PostgreSQL persistence, Alembic migrations, artifact storage, frame-manifest generation, orchestration, evidence storage, API endpoints, and report generation together.

### 19.3 Scientific Validation Tests

Use documented reference examples or synthetic data with known expected outcomes. Test known shifts, flux changes, missing values, coordinate boundaries, WCS behavior, and relevant artifacts.

Define expected tolerances before accepting validation results. Do not update expected values merely to make a failing test pass.

### 19.4 Agent Tests

Test structured outputs, missing evidence, contradictory inputs, provider failures, invalid responses, untrusted external content/prompt injection, and attempts to make unsupported claims or bypass gates.

### 19.5 Public Explorer Acceptance Tests

Test the independent Public Explorer path without creating an investigation or invoking an agent:

- Valid RA/Dec inputs in every supported format.
- Invalid, non-finite, and out-of-range coordinates.
- Supported and unsupported coordinate frames/reference metadata.
- FOV geometry, units, minimum/maximum bounds, and oversized-query rejection.
- Successful queries with zero, one, and multiple observations.
- Provider timeout/failure, malformed metadata, unavailable products, and partial results.
- Frame manifest uses actual observation timestamps for ordering.
- Multiple spectral planes/channels or detector arrays from one epoch are not treated as separate time epochs.
- Frame sequence grouping keeps incompatible releases/products/channels/arrays separate.
- Artifact/cutout/preview references are controlled and do not reveal filesystem paths.
- Missing/invalid WCS or quality metadata is represented honestly.
- Playback, pause, frame stepping, timeline selection, and timestamp display use actual returned frames only.
- Single-frame and zero-frame cases do not generate synthetic frames.
- Blink comparison does not imply astrometric registration or measured motion.
- Public rate, FOV, time-range, frame-count, byte-size, concurrency, and timeout limits are enforced.
- Opening or playing frames does not create an investigation, candidate, or agent task.
- The explicit investigation handoff preserves a server-validated source snapshot when that capability is implemented.

### 19.6 Investigation Acceptance Tests

Run a complete investigation from direct request or explicit Explorer handoff to report. Verify:

- Investigation creation and explicit run creation are separate.
- Capability and prerequisite checks run before dependent analysis.
- Task dependencies, statuses, retries, cancellation, and partial failure behave correctly.
- Deterministic measurements include units, uncertainty, method/version, and provenance where applicable.
- Evidence and candidate assessments are traceable.
- Report claims link to persisted evidence.
- Unsupported, blocked, insufficient-data, and failed outcomes remain distinct.
- Re-analysis preserves earlier result versions.
- Human review is recorded against the reviewed result/evidence version.

### 19.7 NASA Challenge Target Tests

Maintain reproducible target scenarios for Barnard's Star, Epsilon Indi B, WISE 0855, RW Aurigae, and R Draconis. Use the challenge-provided WISE 0855 coordinates. For the other targets, record authoritative coordinate sources and epoch/proper-motion handling where relevant.

The test must verify actual coverage and correct system behavior, not require a positive signal. Record whether each scenario is fixture-only, live-verified, zero coverage, single observation, multiple usable observations, unsupported, or blocked by data limitations. Never label fixture data as live SPHEREx data.

### 19.8 Regression Tests

Maintain fixed fixtures and expected tolerances. Investigate numerical or contract changes rather than blindly updating expected values.

---

## 20. Risks and Controls

| Risk | Control |
|---|---|
| Product becomes an agent dashboard instead of the NASA-required Explorer | Deliver the independent coordinate-to-image-frame Public Explorer path before agent integration. |
| Explorer browsing accidentally starts an investigation | Keep query/frame retrieval separate; require an explicit handoff and separate run request. |
| Dataset does not support motion analysis | Verify product suitability before implementing motion-related claims; return unsupported or insufficient-data outcomes. |
| A particular sky region has no repeated observations | Use actual archive coverage; support zero/one/multiple frames without synthetic epochs. |
| Spectral channels are confused with observation epochs | Keep spectral channel/wavelength, detector array, observation time, and release dimensions separate. |
| Frontend and backend contracts diverge | Use documented, versioned shared contracts and API tests. |
| Agent outputs are inconsistent | Use typed schemas, bounded retries, approved tools, and deterministic scientific functions. |
| Scientific logic is hidden in prompts | Move measurements, tests, and decision rules into versioned code. |
| Results cannot be reproduced | Store provenance, parameters, code/product versions, and input references. |
| Evidence is silently overwritten | Preserve historical runs and use versioned results. |
| Candidate score is mistaken for probability or discovery | Document the ranking policy and calibration limitations; use human review and non-confirmation language. |
| Mandatory gates are bypassed | Enforce gates within the Workflow Orchestrator. |
| Database implementation diverges from the canonical schema | Use PostgreSQL consistently and keep `DATABASE_SCHEMA.md` as the persistence authority. |
| Too much infrastructure too early | Start with a modular monolith and PostgreSQL; add Redis/distributed workers only when justified. |
| Public Explorer endpoints overload the archive or expose artifacts | Enforce FOV, frame, file-size, timeout, concurrency, rate, and controlled-artifact limits. |
| Demonstration implies discovery or live data | Label fixtures and candidate outcomes accurately; claim live support only after archive/product verification. |
| LLM provider failure interrupts scientific integrity | Keep scientific functions, persistence, and evidence validation independent of LLM availability. |
| Three frontend applications become duplicated products | Keep Public Explorer, User Console, and Admin Console separate by responsibility and use shared API contracts/design primitives. |

---

## 21. Recommended First Seven Work Sessions

These are work sessions, not calendar estimates.

1. **Documentation review and repository setup:** Read `MASTER_DEVELOPMENT_PROMPT.md` and review every required project document in its prescribed order; record missing files and conflicts. Create the canonical root structure, three frontend application shells, backend package, root `agents/`, and root `database/` directories.

2. **NASA scope and data feasibility:** Map the challenge statement to P0/P1 requirements. Verify SPHEREx/IRSA product documentation, access routes, release semantics, coordinate/FOV support, actual timestamps, WCS, and availability for the five NASA validation targets. Record unavailable or unverified facts.

3. **Contracts and PostgreSQL foundation:** Finalize Explorer query, observation, frame-manifest, artifact, source-snapshot, investigation, task, evidence, and report contracts. Implement PostgreSQL sessions, SQLAlchemy models, Alembic migrations, and migration tests against PostgreSQL.

4. **Data adapter and validation:** Implement the fixture adapter and the verified IRSA/SPHEREx adapter path where feasible. Validate metadata, WCS, time semantics, release/product identity, quality flags, and zero-result/failure distinctions.

5. **Explorer query and frame manifest:** Implement the P0 coordinate/FOV query, actual observation discovery, frame-manifest generation, controlled cutout/preview delivery, and honest zero/one/multiple-frame behavior.

6. **Public Explorer viewer and NASA target scenarios:** Implement coordinate search, FOV selection, image viewer, timeline, playback, frame stepping, timestamp and metadata display. Test all five target scenarios, keeping fixture results distinct from live archive verification.

7. **First bounded scientific investigation:** Select one analysis supported by verified data. Connect explicit Explorer handoff or direct investigation input to capability checks, deterministic processing, evidence storage, workflow execution, and a traceable report before expanding agent roles.

The actual duration depends on data access, product complexity, and validation requirements.

---

## 22. Release Gate

Do not declare the HELIOS-X MVP ready until both the NASA-facing Explorer path and the bounded investigation path satisfy their applicable acceptance criteria.

### Public Explorer release gate

- [ ] A public visitor can open the Explorer without creating an investigation or signing in.
- [ ] RA/Dec and FOV are validated according to the API contract.
- [ ] The configured data-source adapter returns actual observations/products or a clearly labelled fixture result.
- [ ] The system handles zero, one, and multiple observations honestly.
- [ ] The frame manifest is ordered by actual observation/acquisition timestamps.
- [ ] The frontend displays actual returned frames through controlled artifact delivery.
- [ ] Timeline, play, pause, frame stepping, and active observation-time display work.
- [ ] Product/release/channel and available quality metadata are inspectable.
- [ ] Spectral channels are not treated as time epochs and detector arrays remain a separate dimension.
- [ ] Missing coverage, invalid products, unavailable metadata, unsupported operations, and provider failures remain visible.
- [ ] No fabricated frames or timestamps are presented as SPHEREx data.
- [ ] The five NASA target scenarios have recorded, reproducible outcomes.
- [ ] Public query and artifact resource limits are enforced.

### Agentic investigation release gate

- [ ] One documented, supported investigation completes end to end.
- [ ] Investigation creation and execution start are explicit separate actions.
- [ ] Capability and quality checks run before dependent analyses.
- [ ] Every reported measurement has units, uncertainty where available, method/version, and provenance.
- [ ] Evidence and historical result versions are preserved.
- [ ] Failed dependencies do not produce false success.
- [ ] Unsupported analyses are distinguished from insufficient data.
- [ ] Alternative explanations and limitations are presented.
- [ ] Candidate assessment uses documented deterministic criteria.
- [ ] Reports are evidence-linked and human review decisions are recorded where enabled.
- [ ] Scientific calculations remain independent of LLM availability.
- [ ] No automated confirmation of Planet X or any other new planet is made.

### Integrated platform release gate

- [ ] The Public Explorer remains usable when the agentic engine or LLM provider is unavailable.
- [ ] An explicit Explorer handoff preserves a backend-validated source snapshot.
- [ ] Agents cannot use transient frontend state or client-supplied measurements as authoritative evidence.
- [ ] Public, private, and administrative access boundaries are enforced by the backend.
- [ ] PostgreSQL migrations, backup/restore expectations, artifact references, and operational instructions are documented.
- [ ] All features are labelled accurately as implemented, fixture-only, live-verified, planned, unavailable, or not supported.

---

## 23. Final Implementation Rule

HELIOS-X must be built in the following practical order:

**Structure and document reconciliation → NASA/SPHEREx data feasibility → shared contracts → PostgreSQL persistence → fixture and IRSA data adapters → Public Explorer query/frame-manifest/artifact path → Public Explorer viewer → deterministic scientific methods → evidence and provenance → investigation orchestration → eight logical agents → remaining investigation API → User Console and restricted Admin Console → reports and human review → security/operations → end-to-end and NASA target validation.**

The Public Explorer is the primary NASA-facing deliverable and must work independently of the agentic investigation engine. Do not delay its coordinate-to-observation-to-image-frame experience for advanced scientific methods, agent integration, or optional infrastructure.

The agentic investigation system remains an integrated product capability. It begins only through a direct investigation request or explicit, validated Explorer handoff. The initial objective is not to implement every planned feature or deploy eight separate AI processes. It is to create a reliable public Explorer and one bounded scientific investigation workflow with reproducible measurements, traceable evidence, and honest reporting.

PostgreSQL is the selected relational database. Keep the backend modular, the agent runtime bounded, scientific calculations deterministic, and large artifacts outside relational rows. Additional data sources, agent capabilities, advanced models, and infrastructure should be introduced only when justified by verified scientific requirements and demonstrated need.

**End of Implementation Plan.**
