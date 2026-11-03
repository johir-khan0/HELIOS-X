
# HELIOS-X Operations Guide

**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Document:** `OPERATIONS.md`  
**Version:** 1.1 (MVP — architecture-aligned)  
**Status:** Developer and local-operations guide  
**Backend:** Python / FastAPI modular monolith  
**Frontends:** React / TypeScript / Vite (`frontend/`, `user-console/`, `admin-console/`)  
**Database:** PostgreSQL, managed through SQLAlchemy 2.x and Alembic  

---

## 1. Purpose

This document explains how a developer should configure, run, test, debug, migrate, and recover the HELIOS-X MVP in a local environment.

HELIOS-X has two connected but independently usable capabilities:

1. **Public SPHEREx Sky Image Explorer:** a public, read-only experience for coordinate-based sky queries, observation discovery, frame timelines, image playback, and related metadata.
2. **Agentic Scientific Investigation Engine:** an explicitly initiated workflow for scientific analysis, evidence, candidate assessment, reports, and human review.

The repository has three separately buildable frontend applications:

- `frontend/` — Public Explorer.
- `user-console/` — authenticated user and investigation workspace.
- `admin-console/` — restricted administrative interface, only for operational features that are actually implemented.

The backend is one FastAPI modular monolith. The eight agent roles are logical components in the root-level `agents/` package; they are not eight separately deployed services. The MVP does not require microservices, Kubernetes, Kafka, Celery, or Redis. Redis may be considered later only if a demonstrated requirement justifies it; it is not the system of record.

PostgreSQL is the selected application database for local development, CI/integration, staging, and production. The canonical schema and database design are documented in `database/DATABASE_SCHEMA.md`. Large FITS files, image arrays, spectra, previews, cutouts, and report binaries belong in controlled artifact storage, not in PostgreSQL rows.

Commands below are intended baselines. Exact package names, scripts, environment-variable names, and paths must be reconciled with the repository as implementation proceeds. This guide does not claim that commands or endpoints have already been implemented or verified.

## 2. Prerequisites

Install the following:

- Git.
- A supported Python version recorded in the backend project configuration.
- Node.js LTS and npm (or the package manager selected by the repository).
- PostgreSQL, using the version selected and documented by the project.
- A code editor and terminal.
- Optional: Docker and Docker Compose, if container execution is adopted.

Use the runtime versions pinned or documented by the repository. Do not assume a version that differs from `pyproject.toml`, lock files, or project setup documentation.

Redis is not a prerequisite for the MVP. Do not add it to local setup merely because it may be useful at a later scale stage.

## 3. Repository and Environment Setup

### 3.1 Clone and enter the repository

```bash
git clone <repository-url>
cd <repository-directory>
```

Replace the placeholders with the actual repository URL and directory.

### 3.2 Create backend virtual environment

Windows PowerShell:

```powershell
py -m venv .venv
.\\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

macOS/Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

Install backend dependencies using the project's chosen, locked dependency workflow. For example, if `pyproject.toml` defines the project and development extras:

```bash
pip install -e ".[dev]"
```

Do not use this example until the corresponding package configuration exists. Install the PostgreSQL driver selected by the project; the current database decision is PostgreSQL with SQLAlchemy 2.x, Alembic, and `psycopg`.

### 3.3 Install frontend dependencies

Install dependencies separately for each frontend that exists in the repository and has a committed lock file:

```bash
cd frontend
npm ci
cd ..

cd user-console
npm ci
cd ..

cd admin-console
npm ci
cd ..
```

A frontend that has not yet been implemented does not need to be installed or started. `npm ci` requires a committed, current lock file. Use the package manager and lock file selected for each application.

### 3.4 PostgreSQL setup

Start a local PostgreSQL instance and create a database and application role for HELIOS-X. The runtime application role should have only the permissions needed by the application; it should not be a PostgreSQL superuser.

Keep migration privileges separate from ordinary application-runtime privileges where the project configuration supports it. Do not expose PostgreSQL directly to frontend applications or public clients.

The exact PostgreSQL version, role-creation procedure, and connection settings must match the project's setup and deployment configuration. Do not place database credentials in source control.

### 3.5 Environment configuration

Create a local `.env` from `.env.example` when that example exists. Do not commit `.env`.

Illustrative development settings may include:

```dotenv
APP_ENV=development
API_HOST=127.0.0.1
API_PORT=8000
DATABASE_URL=postgresql+psycopg://helios_app:<password>@127.0.0.1:5432/helios_x
LOG_LEVEL=INFO
FRONTEND_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost:5175
ARTIFACT_STORAGE_ROOT=./artifacts
DATA_ROOT=./data
```

These names and port assignments are proposed examples; the implementation must define and validate the actual settings. URL-encode special characters in database connection credentials as required by the connection URL format.

Store provider credentials only if the selected data adapter requires them. Keep archive, catalogue, LLM-provider, database, and artifact-storage credentials on the backend. Never put secret values in frontend environment variables that are bundled into browser code. Frontend environment variables may contain public configuration such as the API base URL, but not credentials.

### 3.6 Local data and artifact directories

Keep the PostgreSQL data directory, downloaded observations, generated artifacts, reports, and logs outside frontend public/static directories. A proposed local layout is:

```text
data/
  fixtures/                 # small, clearly labelled test fixtures only
artifacts/
  observations/             # retrieved source products, where retained
  previews/                 # generated display previews and cutouts
  spectra/                  # generated or retrieved spectral artifacts
  reports/                  # generated report files
logs/
```

PostgreSQL manages its own database files; do not create or copy a SQLite database for this application. The actual paths should be configurable. Add local data, artifact files, secrets, logs, database dumps, and large retrieved products to `.gitignore` unless a small test fixture is intentionally version-controlled.

The database stores artifact metadata and controlled references. It does not store large FITS files, image arrays, spectra, or report binaries as ordinary relational rows.

---

## 4. Start the Application

### 4.1 Start the backend

From the repository root, with the backend environment active:

```bash
uvicorn helios_x.api.main:app --app-dir backend/src --reload --host 127.0.0.1 --port 8000
```

This is the expected command if the backend package follows the established `backend/src/helios_x/` structure and the API entry point is `helios_x.api.main:app`. Adjust it if implementation uses a different entry point. Bind to `127.0.0.1` during local development unless external access is required and secured.

Common local URLs, if the corresponding routes are implemented:

- API base: `http://127.0.0.1:8000/api/v1`
- OpenAPI UI: `http://127.0.0.1:8000/docs`
- OpenAPI schema: `http://127.0.0.1:8000/openapi.json`

Do not expose interactive API docs publicly without reviewing deployment access controls.

### 4.2 Start the Public Explorer

In a separate terminal:

```bash
cd frontend
npm run dev
```

Use the URL printed by Vite. A common development default is `http://localhost:5173`. Configure the API base URL through the frontend's documented public environment setting.

The Public Explorer may call only the designated public, read-only API operations. Browsing, coordinate lookup, timeline navigation, playback, blinking, and frame inspection must not create an investigation or start an agent run.

### 4.3 Start the User Console

If `user-console/` has been implemented, start it in a separate terminal:

```bash
cd user-console
npm run dev
```

Use the URL printed by Vite and configure its API base URL through the documented frontend setting. Private investigation resources must be protected by backend authentication and resource-level authorization; frontend routing or hidden controls are not security boundaries.

### 4.4 Start the Admin Console

If `admin-console/` has been implemented and there is an approved operational use for it, start it in a separate terminal:

```bash
cd admin-console
npm run dev
```

The Admin Console is restricted. Do not expose administrative API operations to anonymous users. Do not create placeholder controls that imply an administrative operation exists when its backend behavior has not been implemented.

### 4.5 Verify startup

Use the actual endpoints implemented by the API specification. The intended minimum is:

```bash
curl http://127.0.0.1:8000/api/v1/health
curl http://127.0.0.1:8000/api/v1/ready
```

A capabilities endpoint may also be used if it has been implemented, for example:

```bash
curl http://127.0.0.1:8000/api/v1/capabilities
```

If an endpoint has not yet been implemented, do not interpret a `404` as a server health failure; track the missing endpoint as unfinished work. A healthy API process does not prove that every external archive, catalogue, or analysis capability is available.

---

## 5. Configuration Rules

Configuration should be loaded once at application startup and validated. Avoid scattered hard-coded values.

Document and validate:

- Environment name.
- API host/port.
- PostgreSQL database URL and required driver.
- Data, artifact, and report storage directories.
- Log level.
- Allowed frontend/CORS origins for all implemented frontends.
- Approved provider endpoints and credentials, if needed.
- Public Explorer query, response, file, and processing limits.
- Artifact download and processing limits.
- Workflow retry and timeout limits, where configured.

Fail at startup if a required setting is missing or invalid. Error messages must identify the setting name without printing its secret value.

Do not make Redis, a message broker, a separate agent service, or a container platform mandatory unless a later approved architecture change establishes that requirement.

---

## 6. PostgreSQL Initialization and Migrations

PostgreSQL is the selected application database for local development, CI/integration, staging, and production. SQLite is not the application database or an automatic fallback. The canonical schema and database decisions are documented in `database/DATABASE_SCHEMA.md`.

### 6.1 Database location and access

Use the configured PostgreSQL instance and application database. Ensure the database is reachable by the backend and is not directly exposed to browser clients. Keep database credentials in environment/secret configuration.

The application runtime role must not be a superuser. Use the approved migration process for schema changes; do not give agents or ordinary frontend/API users database-administration privileges.

### 6.2 Schema migrations

SQLAlchemy 2.x models and Alembic migrations define the schema. Once Alembic is configured, the intended commands are:

```bash
alembic upgrade head
alembic current
alembic history
```

Run migrations from the directory and environment where the project's Alembic configuration is available. The migration path is `backend/migrations/`; use the repository's actual Alembic configuration and command options if they differ from the examples above.

When a migration changes schema:

1. Review the migration script.
2. Back up any database containing data that must be preserved.
3. Apply the migration using the approved migration credentials/process.
4. Run persistence and integration tests.
5. Verify the application can read and write expected resources.

Do not rely on automatic table creation as a replacement for versioned migrations in a shared or persistent environment.

### 6.3 PostgreSQL safeguards

- Use the PostgreSQL runtime role with the minimum permissions required by the application.
- Keep migration privileges separate from ordinary runtime privileges where practical.
- Use transactions for related writes and keep transactions short.
- Preserve previous task attempts and scientific results where auditability requires it.
- Do not store large FITS files, image arrays, spectra, previews, or report binaries as database blobs; store controlled artifact references, checksums, and metadata.
- Use the repository/persistence layer rather than ad hoc database edits.
- Do not manually edit production or shared database records outside a documented recovery procedure.
- Keep remote PostgreSQL access restricted and use TLS when connections cross untrusted networks.

Redis is optional and is not a durable system of record. The MVP must work without Redis unless a later approved implementation explicitly adds a required use case.

---

## 7. Run Tests

Run commands from the relevant project directory. Use the test commands and scripts actually configured in the repository.

### 7.1 Backend tests

```bash
pytest
```

If test markers are configured, useful focused commands may include:

```bash
pytest -m unit
pytest -m integration
pytest -m scientific
```

These markers are optional until configured. Unit tests should not require network access. External data-provider calls should be mocked in core CI tests. Persistence and integration tests should run against PostgreSQL so the tested database behavior matches the selected application database.

### 7.2 Frontend tests

Use the scripts declared in each implemented frontend's `package.json`, for example:

```bash
cd frontend
npm test
npm run build
```

Repeat the applicable test/build commands for `user-console/` and `admin-console/` when those applications have been implemented. These commands require corresponding scripts to exist. Record the actual test runner and commands in the repository's developer documentation once selected.

### 7.3 End-to-end tests

The MVP should have repeatable tests for both product capabilities. Use a supported dataset or a clearly labelled sample dataset.

**Public Explorer test should verify:**

- A supported RA/Dec and FOV query returns the expected supported observation results.
- The frame manifest uses actual observation timestamps and is ordered by those timestamps.
- Frame navigation/playback uses available temporal observations, not spectral channels as separate time epochs.
- Missing observations are not fabricated to create a continuous sequence.
- Public browsing and playback remain read-only and do not create investigations or trigger agent execution.
- Unsupported inputs and configured query/file/result limits are handled correctly.

**Investigation workflow test should verify:**

- Explicit investigation creation or handoff from a validated Explorer selection.
- Workflow start and status retrieval.
- At least one deterministic scientific result.
- Evidence/provenance retrieval.
- Report generation and retrieval.
- Correct handling of at least one failure, unavailable product, or insufficient-data case.

Core automated tests must not fail merely because a live data provider is temporarily unavailable. Provider behavior should be mocked in ordinary CI tests; live-provider checks should be separate and intentional.

---

## 8. Logging and Debugging

### 8.1 Logging

Use structured application logs with configurable levels:

- `DEBUG` for local troubleshooting.
- `INFO` for ordinary lifecycle events.
- `WARNING` for recoverable abnormal conditions.
- `ERROR` for failed operations.

Include request ID and, where applicable, investigation ID, execution ID, task ID, attempt number, component/agent role, artifact ID, and method version. Record useful provider outcomes and public endpoint failures without logging full raw scientific payloads.

Never log secrets, access tokens, full authorization headers, database connection strings, or unnecessary large raw scientific payloads. Sanitize untrusted values before writing them to logs.

### 8.2 Debugging Explorer and data-retrieval issues

When a sky query or image frame fails:

1. Check the request/correlation ID and API response status.
2. Verify the coordinate, FOV, time/filter parameters, and public query limits.
3. Check the capability response and whether the relevant provider/product is supported.
4. Inspect the archive adapter logs and bounded provider error summary.
5. Verify that the returned observation/product reference is valid and that the artifact exists.
6. Check the actual observation timestamp and product provenance.
7. Do not fabricate a frame or substitute a spectral channel for a temporal observation.

### 8.3 Debugging investigation workflow failures

When an investigation fails:

1. Open its execution details in the User Console, if implemented.
2. Identify the failed or blocked task.
3. Inspect the task's safe error summary and request/correlation ID.
4. Check backend logs for that ID.
5. Verify task input, dependency statuses, quality outcomes, and relevant data-source response.
6. Retry only if the orchestrator/API says retry is permitted.
7. Preserve the failed attempt and its evidence for diagnosis.

Do not manually mark a failed task as successful to make the UI proceed. Do not bypass deterministic scientific checks or quality gates to force report generation.

### 8.4 Common development problems

| Symptom | Check |
|---|---|
| Backend command cannot import the app | Confirm virtual environment, package installation, `backend/src` path, and actual module path |
| A frontend cannot call the API | Check that frontend process, API base URL, allowed CORS origin, and browser network errors |
| PostgreSQL connection fails | Check PostgreSQL process, database name, role permissions, connection URL, and network access |
| Schema/model mismatch | Check Alembic migration status and run persistence tests |
| Data retrieval times out | Check provider availability, approved endpoint, timeout settings, network access, and adapter logs |
| Explorer has no frames | Check supported coverage, actual observation timestamps, product availability, and query limits |
| Workflow appears stuck | Inspect task status, active attempt, dependencies, timeout, and orchestrator logs |
| Report is not available | Check report-generation task status and whether the requested format is implemented |
| Artifact reference exists but file is missing | Check artifact storage location, metadata, checksum, and recovery status |
| Test unexpectedly calls the network | Replace provider calls with mocks or mark the test as an explicit live-data test |

---

## 9. Health and Readiness

Use separate checks for separate purposes:

- **Liveness (`/health`):** the API process is running and can answer a simple request.
- **Readiness (`/ready`):** required local dependencies, including PostgreSQL where required, are sufficiently available for the application to serve work.
- **Capabilities (`/capabilities`):** the current deployment's supported data sources, products, and analysis operations.

Health endpoints must not trigger scientific analysis. They must not expose secrets, full configuration, local paths, or detailed internal exception text.

A healthy API process does not mean every external provider or scientific capability is available. Display capability/provider availability separately where relevant. Do not make a temporary external archive outage appear to be a database or API-process failure unless the application explicitly requires that provider for readiness.

---

## 10. Backup and Recovery

### 10.1 What to protect

Back up as appropriate:

- PostgreSQL database.
- Required configuration, without putting secrets into an unprotected backup.
- Important retrieved observation files or their reproducible retrieval/checksum metadata.
- Generated reports, previews, spectra, and evidence artifacts that must be retained.
- Fixture/version metadata required to reproduce an investigation.

The database and artifact storage are related: a restored database may reference artifacts that must also be available. Maintain a clear backup/manifest relationship so restoration does not silently produce broken artifact references.

### 10.2 Local MVP backup

Before migrations, destructive tests, or major changes, make a consistent PostgreSQL backup. Use PostgreSQL's supported backup tools or a documented application/deployment command; do not copy the live PostgreSQL data directory as an ordinary file backup.

For example, a custom-format logical backup may be created with the PostgreSQL `pg_dump` utility:

```bash
pg_dump -Fc -h 127.0.0.1 -U <backup-role> -d helios_x -f helios_x.dump
```

The database name, host, role, authentication method, and destination must match the local configuration. Do not put passwords directly in shell history or committed scripts. This example does not back up artifact files; protect required artifacts separately and keep a matching manifest or documented recovery procedure.

Keep backups outside the repository's public/static directories. Do not commit database dumps or private artifacts to Git. Restrict backup access and encrypt backups where the storage environment supports it.

### 10.3 Recovery procedure

1. Stop the backend to prevent additional application writes.
2. Preserve the damaged database state and relevant logs for diagnosis where practical.
3. Restore the latest known-good PostgreSQL backup to the configured database using the approved restore procedure (for a custom-format dump, `pg_restore` may be used).
4. Confirm schema/migration version.
5. Restore or verify referenced observation and report artifacts; compare checksums where available.
6. Start the backend and run health/readiness checks.
7. Run persistence and workflow smoke tests.
8. Record missing or unrecoverable artifacts and do not silently present them as available.

Do not restore over a database containing data that must be preserved without first making a separate backup and confirming the target.

### 10.4 MVP recovery expectations

The MVP must document how to back up and restore the PostgreSQL database and how to identify missing artifacts. Perform at least one restore test before relying on a backup. Automated scheduled backups, point-in-time recovery, multi-region replication, and formal disaster-recovery infrastructure are not required for a local MVP unless the deployment context requires them.

---

## 11. Docker-Based Execution

Docker is optional for the initial local MVP. Add it when it makes setup repeatable across contributors or CI; do not introduce it as a prerequisite solely for appearance.

If Docker is adopted:

- Provide a `Dockerfile` for the required application process(es).
- Provide a minimal Compose configuration only if multiple services need coordinated startup, such as the backend and PostgreSQL.
- Persist PostgreSQL data and required artifacts in named volumes or clearly defined bind mounts.
- Use environment variables or a supported secret mechanism for configuration and secrets.
- Bind development ports deliberately.
- Avoid running as root where practical.
- Do not bake secrets, database dumps, or local data into images.
- Provide documented migration and backup commands for containerized PostgreSQL and artifact data.
- Add a health check only when its target endpoint is implemented.
- Pin base image versions and rebuild images when security updates are required.

Do not add a separate container or deployment for every logical agent. The MVP agent roles run within the application architecture unless a later measured requirement justifies a separate execution boundary.

The Docker workflow must preserve the same tests and data-integrity expectations as local execution.

---

## 12. Data and Artifact Management

- Keep PostgreSQL data, production-like data, downloaded products, reports, and logs outside source code directories used for static serving.
- Store source product references, actual observation timestamps where available, checksums, provenance, and required metadata with scientific outputs.
- Keep large FITS files, image arrays, spectra, previews, cutouts, and report binaries in controlled artifact storage, not in PostgreSQL rows.
- Do not silently overwrite reports or results that need version history.
- Label fixtures and demonstration datasets clearly.
- Document storage paths and cleanup behavior.
- Remove temporary files after failed processing when safe, without deleting artifacts needed to diagnose a failure.
- Do not assume a file exists merely because its database reference exists; handle missing files clearly.
- Check authorization before delivering private artifact metadata or content. Public Explorer delivery is limited to artifacts explicitly designated for public access.
- Do not expose arbitrary filesystem paths or unrestricted provider URLs through artifact APIs.

---

## 13. MVP Operational Acceptance Checklist

Before accepting the operational setup:

- [ ] A new developer can follow the documented setup instructions.
- [ ] Backend and each implemented frontend start using documented commands.
- [ ] PostgreSQL is configured as the application database; SQLite is not used as an application fallback.
- [ ] Configuration is validated and secrets are not committed.
- [ ] Health, readiness, and capability behavior is documented and testable where implemented.
- [ ] PostgreSQL initializes and changes through versioned Alembic migrations.
- [ ] Application runtime database permissions are limited; frontend applications do not connect directly to PostgreSQL.
- [ ] Backend and implemented frontend tests have documented commands.
- [ ] Public Explorer query, frame-manifest, playback, and artifact-delivery behavior is tested with supported or clearly labelled sample data.
- [ ] Explorer frame timelines use actual observation timestamps; spectral channels are not represented as separate temporal epochs.
- [ ] Public browsing and playback do not create investigations or start agent runs.
- [ ] An explicit, validated handoff can start the investigation workflow where implemented.
- [ ] One supported or labelled sample investigation-to-report workflow can be run again.
- [ ] Logs include useful request/execution identifiers without exposing secrets or unnecessary raw data.
- [ ] A failed task can be diagnosed from its status, safe error summary, and logs.
- [ ] PostgreSQL backup and restore instructions have been tested at least once.
- [ ] Required artifacts can be matched to database references and checked for missing files.
- [ ] Docker is documented only if the repository includes a supported Docker workflow.
- [ ] Known limitations and provider prerequisites are recorded.

---

## 14. Recommended Operational Runbook

For an ordinary development session:

1. Pull the intended code revision.
2. Activate the project's Python environment and install locked backend dependencies when they change.
3. Install frontend dependencies when a relevant lock file changes.
4. Confirm local PostgreSQL is running and validate environment configuration.
5. Apply pending database migrations through the configured Alembic process.
6. Start the FastAPI backend.
7. Start the frontend application(s) needed for the work: Public Explorer, User Console, and Admin Console only when implemented and required.
8. Check API readiness and the capability list.
9. Run the relevant automated tests before making a change.
10. Reproduce and inspect failures using request, execution, and task IDs.
11. Run tests again after the change.
12. Back up any local database and required artifacts that must be preserved before risky migrations or cleanup.
13. Record any new setup step or known limitation in the developer documentation.

---

## 15. Items to Finalize During Implementation

1. Exact supported Python, Node.js, and PostgreSQL versions.
2. Final dependency-locking and installation commands.
3. Actual backend module path and package manager scripts for all implemented frontends.
4. Exact environment variable names and defaults.
5. Alembic configuration and migration command/location.
6. Actual health, readiness, and capability endpoints.
7. Test runner commands and available markers.
8. Local data/artifact directories, backup procedure, and retention rules.
9. Approved archive/catalogue endpoints and provider prerequisites.
10. Whether Docker is needed for contributor onboarding or CI.

Do not document an optional service or command as required until it exists in the repository. Keep this guide aligned with the actual implementation and the canonical architecture/database documents; do not duplicate their detailed specifications here.

---

**End of document.**
