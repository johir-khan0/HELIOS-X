
# HELIOS-X Testing and Acceptance

**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Document:** `TESTING_AND_ACCEPTANCE.md`  
**Version:** 1.1 (MVP — architecture-aligned)  
**Status:** MVP testing and acceptance specification  
**Backend:** Python / FastAPI modular monolith  
**Frontends:** React / TypeScript / Vite  
**Database:** PostgreSQL with SQLAlchemy 2.x and Alembic  
**Test tools:** pytest for backend; frontend test tooling selected during implementation

---

## 1. Purpose

Define the minimum tests required to show that the HELIOS-X MVP works correctly, handles failures safely, preserves scientific integrity, and provides the intended user experience.

HELIOS-X has two connected but independently usable capabilities:

1. **Public SPHEREx Sky Image Explorer:** coordinate-based sky queries, observation discovery, actual observation timelines, image-frame navigation, playback, and metadata inspection.
2. **Agentic Scientific Investigation Engine:** explicitly initiated investigations, deterministic scientific analysis, workflow execution, evidence/provenance, candidate assessment, reports, and human review where implemented.

The project has three separately buildable frontend applications:

- `frontend/` — Public Explorer.
- `user-console/` — authenticated investigation workspace.
- `admin-console/` — restricted administrative interface, limited to operational features actually implemented.

The backend is a single FastAPI modular monolith. The eight agent roles are logical components, not eight independently deployed services.

This document defines acceptance criteria. It does not mean that the tests have already been implemented or passed.

The implementation phase order is owned by `IMPLEMENTATION_PLAN.md`. Runtime, setup, test execution commands, and recovery procedures are owned by `OPERATIONS.md`. This document defines what must be tested and what evidence is required for acceptance.

---

## 2. MVP Testing Scope

The MVP requires the following test groups:

1. **Unit tests** — input validation, schemas, status transitions, deterministic calculations, and result formatting.
2. **Integration/API tests** — FastAPI routes, Workflow Orchestrator, scientific services, PostgreSQL persistence, and controlled artifact storage.
3. **Scientific validation tests** — deterministic analysis checked against known-answer synthetic data.
4. **Public Explorer tests** — coordinate queries, observation results, frame manifests, timestamps, playback, and controlled image delivery.
5. **Frontend tests** — Public Explorer and User Console workflows, including loading, empty, error, and insufficient-data states. Admin Console tests apply only to implemented functionality.
6. **End-to-end tests** — the Public Explorer journey and one supported investigation-to-report journey, using supported or clearly labelled sample data.
7. **Security and failure tests** — important input, permission, artifact-access, failure-propagation, and information-exposure boundaries.

Separate large-scale performance testing, extensive LLM benchmarking, and exhaustive live-provider testing are not MVP release requirements unless the selected deployment or data source makes them necessary.

Core automated tests must not depend on an external archive, catalogue, or LLM provider being continuously available.

---

## 3. Test Environment and Database

### 3.1 Database

PostgreSQL is the selected application database for local development, CI/integration, staging, and production.

Use PostgreSQL for persistence and integration tests. SQLite is not the application database or an automatic test fallback.

Tests must use an isolated PostgreSQL test database or an equivalent isolated test environment. Never run destructive tests against a production or shared database.

### 3.2 Persistence and migrations

Test:

- Alembic migrations apply successfully to a clean PostgreSQL test database.
- The schema can be upgraded from the previous supported migration state.
- SQLAlchemy models and database constraints match `database/DATABASE_SCHEMA.md`.
- Foreign-key and other configured constraints behave correctly.
- Related writes use transactions and roll back when required.
- Records remain available after closing and reopening the database connection/session.
- Tests do not depend on data left behind by previous test runs.

### 3.3 Artifact storage

Large FITS files, image arrays, spectra, previews, cutouts, and report binaries belong in controlled artifact storage, not ordinary PostgreSQL rows.

Use an isolated temporary artifact-storage location or test adapter where appropriate.

Verify:

- Artifact metadata and database references are consistent.
- A valid artifact can be retrieved through the approved application interface.
- Missing or corrupt artifacts are handled explicitly.
- An artifact cannot be accessed through an arbitrary filesystem path.
- Artifact content and metadata access follow the relevant public/private access rules.

The test environment does not require a production object-storage service unless a selected implementation specifically depends on one.

---

## 4. Minimum Test Data

Create a small, version-controlled set of synthetic fixtures with known expected outcomes.

Each fixture should record:

- Fixture ID and version.
- Purpose and expected outcome.
- Relevant units and metadata.
- Expected value and method-specific tolerance, where applicable.
- Whether the fixture represents synthetic/demo data.
- Relevant observation timestamps and source/product references where applicable.

### 4.1 Scientific and workflow fixtures

| ID | Test case | Expected outcome |
|---|---|---|
| `SYN-001` | Valid observation/product metadata | Accepted and normalized |
| `SYN-002` | Missing required input or metadata | Clear validation failure or `INSUFFICIENT_DATA` |
| `SYN-003` | Corrupt or unreadable input file | Explicit processing failure; no fabricated result |
| `SYN-004` | Synthetic data with a known measurable source/value | Result matches the expected value within a method-specific tolerance |
| `SYN-005` | Known catalogue match and known no-match | Correct match/no-match result; no-match is not treated as a discovery |
| `SYN-006` | Workflow task failure or unavailable dependency | Failure propagates; dependent tasks do not falsely succeed |
| `SYN-007` | Cancellation or repeated run request | Documented cancellation/idempotency behavior is respected |
| `SYN-008` | Unsupported analysis request | Explicitly rejected or marked `NOT_SUPPORTED` |

Only create fixtures for scientific methods actually included in the MVP. For example, if motion analysis is not part of the first supported workflow, its specialized synthetic dataset is not a release requirement.

### 4.2 Public Explorer fixtures

| ID | Test case | Expected outcome |
|---|---|---|
| `SKY-001` | Valid supported RA/Dec query | Correctly validated and normalized coordinate request |
| `SKY-002` | Query with available observations | Returns only valid supported observation/product records |
| `SKY-003` | Multiple observations at distinct timestamps | Frame manifest is ordered by actual observation timestamps |
| `SKY-004` | Multiple spectral channels in one observation | Spectral channels are represented as wavelength samples, not separate time epochs |
| `SKY-005` | Successful query with no matching observations | Clear empty/no-result outcome; not represented as a provider failure or proof that no object exists |
| `SKY-006` | Provider timeout or failed archive request | Explicit retrieval failure; not represented as a successful empty result |
| `SKY-007` | Missing, corrupt, or unavailable image artifact | Clear unavailable/error state; no fabricated frame |
| `SKY-008` | Unsupported coordinates, FOV, time range, or excessive request | Validation or configured-limit response |
| `SKY-009` | Frame navigation/playback across observations | Uses available actual observation frames and does not fabricate missing epochs |
| `SKY-010` | Public Explorer browsing and playback | Does not create an investigation, candidate, or agent execution |

Use fixtures with known observation timestamps to verify timeline behavior. Do not generate artificial observation dates merely to make a timeline appear continuous.

---

## 5. Scientific Validation

Every scientific method included in the MVP must have at least one deterministic known-answer test.

For each method, record:

- Input fixture and version.
- Expected result and units.
- Absolute or relative tolerance, with a short rationale.
- Required uncertainty and quality metadata.
- Method/software version.
- Known limitations.

Choose tolerances for each method; do not use one universal tolerance for every calculation. Define the expected result and tolerance before running the test.

### 5.1 Required scientific safeguards

- Scientific measurements come from tested deterministic code, not invented LLM output.
- Missing data must not be treated as a detection or a zero-valued measurement.
- A catalogue no-match must not be treated as proof of a new astronomical object.
- SPHEREx's 102 spectral channels must not be treated as 102 time epochs.
- Temporal playback must use actual distinct observation timestamps.
- Missing observations must not be fabricated or silently interpolated into a false observation history.
- Failed quality checks and uncertainty limitations must remain visible.
- Unsupported or insufficient-data analysis must not return a fabricated measurement.
- Demo/sample results must be labelled as such.
- Derived results must retain source references, method/version, units, uncertainty where available, and relevant quality information.
- An LLM must not invent source IDs, observation timestamps, measurements, uncertainty values, units, or catalogue matches.

### 5.2 Method-specific scope

Only test scientific methods that the current MVP declares supported.

A method that is planned but not implemented must not be presented as passed. A method that lacks required input data must return the applicable documented outcome, such as `NOT_SUPPORTED` or `INSUFFICIENT_DATA`.

---

## 6. Public Explorer Tests

The Public Explorer is the primary public-facing SPHEREx experience and requires its own acceptance tests. It must not be treated as merely another investigation screen.

### 6.1 Coordinate query

Test that:

- Valid RA/Dec inputs are accepted according to the API contract.
- Invalid or non-finite coordinates are rejected.
- Supported FOV and query constraints are enforced.
- Unsupported query parameters are not silently accepted.
- Query results preserve observation/product references and available metadata.
- Empty, failed, and unavailable-data outcomes are distinguishable.

### 6.2 Observation discovery and frame manifest

Test that:

- The returned frame manifest contains only valid supported observation/product references.
- Actual observation timestamps are present where supplied by the source.
- Frames are ordered using their actual observation timestamps.
- Missing timestamps or incomplete product metadata are handled explicitly.
- Multiple spectral channels belonging to an observation are not represented as separate temporal frames.
- The UI does not invent observation dates or silently fill coverage gaps.
- The source observation/product relationship remains traceable.

### 6.3 Frame viewer and playback

Test that:

- A valid frame can be opened and inspected.
- Previous/next navigation follows the returned frame sequence.
- Playback advances only through available temporal observation frames.
- Pausing and stepping through frames do not change the underlying observation identity.
- Missing or failed frame retrieval is shown clearly.
- A failed image request does not silently display an unrelated or fabricated frame.
- Image metadata and observation timestamps remain associated with the correct frame.
- The viewer communicates when no valid temporal sequence is available.

### 6.4 Public/private boundary

Test that:

- Public Explorer operations remain limited to designated read-only capabilities.
- Anonymous browsing and frame inspection do not initiate an investigation or agent run.
- Private investigation resources are not exposed through public Explorer routes.
- Artifact delivery exposes only artifacts designated for public access.
- Configured query, response-size, frame-count, image-size, timeout, and rate limits are enforced.

An explicit user action is required to hand selected Explorer context to the investigation workflow.

Where this handoff is implemented, test that:

- The selected coordinates, relevant observation/product references, and supported context are transferred.
- The backend validates the submitted selection instead of blindly trusting browser state.
- The validated selection is associated with the created investigation.
- A failed handoff does not display a successful investigation state.
- Ordinary Explorer browsing or playback never starts an investigation implicitly.

---

## 7. Core Investigation Workflow and API Tests

Test the following supported workflow:

1. Create an investigation.
2. Retrieve its details.
3. Start a workflow and receive an execution ID.
4. Retrieve execution/task status.
5. Run at least one supported deterministic analysis.
6. Retrieve the result/evidence with available provenance and quality information.
7. Request report generation.
8. Retrieve the completed report.
9. Confirm that the report includes methods, source/data references, uncertainty where available, and limitations.

Where Explorer handoff is implemented, also verify that a validated Explorer selection can be used to start an investigation without losing its source context.

### 7.1 API behavior

Also test:

- Invalid request body returns a documented validation error.
- Invalid coordinate, time, FOV, or filter values are rejected where applicable.
- Unknown resource ID returns the documented not-found outcome.
- Invalid state transition is rejected.
- Failed task is not shown as successful.
- Cancellation and retry follow the documented rules.
- Duplicate run requests follow the documented idempotency behavior.
- A report that is not ready is not returned as successfully completed.
- Missing and unavailable resources are represented accurately.
- OpenAPI matches the implemented API schemas and the documented public contract.
- API responses do not expose database rows or internal persistence details that are not part of the contract.

Tests should use the canonical resource IDs, status values, error codes, and request/response contracts defined by the API and workflow specifications.

---

## 8. Frontend Tests

Test the three frontend applications according to their implemented responsibilities.

### 8.1 Public Explorer (`frontend/`)

Verify that a user can:

- Enter and validate supported sky coordinates.
- Submit a supported query.
- Inspect returned observations and product metadata.
- View the actual observation timeline.
- Navigate between available frames.
- Play, pause, and step through supported temporal frames, where implemented.
- Understand when no observations or usable temporal frames are available.
- Distinguish successful empty results from retrieval failures.
- See loading, validation, unavailable-artifact, and API-error states.
- Explicitly hand selected context to the investigation workflow, if implemented.

The UI must not show fake observation frames, fabricated coverage, or false six-month timelines.

### 8.2 User Console (`user-console/`)

Verify that an authorized user can:

- Open the Workspace.
- Create and open an investigation.
- Start a supported run.
- See execution and task status from the backend.
- See loading, empty, failed, cancelled, blocked, unsupported, and insufficient-data states.
- Open results, evidence, and candidate assessments.
- Retrieve a completed report.
- Understand uncertainty, quality outcomes, and scientific limitations.
- Recover from a temporary API error without the UI showing false success.
- Refresh a private resource page and retrieve its current state from the backend.

The frontend must not invent persistent scientific state or bypass backend permissions.

### 8.3 Admin Console (`admin-console/`)

The Admin Console is a separate restricted application. It is not a reason to introduce unnecessary administrative features into the MVP.

If administrative functionality is implemented, verify that:

- Administrative routes require the appropriate backend authorization.
- Unauthorized users cannot access administrative operations by directly calling the API.
- Only implemented and approved operational controls are displayed as usable.
- Configuration or operational changes are validated and auditable where applicable.
- Administrative actions cannot bypass scientific quality gates or rewrite source observations and deterministic measurements.

Do not require an unimplemented Admin Console feature to pass MVP acceptance.

### 8.4 General frontend behavior

Use component tests for important shared controls and screen states. Use automated browser-based end-to-end testing where browser automation is configured for the MVP.

Verify:

- Keyboard navigation and visible focus.
- Dark and light theme readability.
- Responsive behavior at agreed viewport sizes.
- Status labels and semantic colors.
- Form validation and preservation of entered values after recoverable errors.
- No duplicate mutations from repeated clicks.
- Accessible text alternatives for meaningful charts and image information.

---

## 9. Failure, Persistence, and Security Tests

### 9.1 Workflow and persistence

Using PostgreSQL and controlled mocks, verify:

- Failed writes roll back correctly.
- Database constraints behave as documented.
- Data remains available after reopening the database connection/session.
- Alembic migrations apply to a clean test database.
- A failed data retrieval or unavailable provider does not make the workflow appear successful.
- Retry does not silently overwrite previous attempts or scientific results.
- Duplicate run requests follow the documented idempotency behavior.
- Partial workflow failure is represented accurately.
- Dependent tasks remain blocked when their required prerequisites fail.
- Reports do not claim completion when required work failed or remained incomplete.

### 9.2 Artifact handling

Verify:

- Large scientific artifacts are stored outside PostgreSQL relational rows.
- Valid artifact IDs retrieve the intended artifact.
- Missing artifacts return a clear unavailable/error outcome.
- Artifact IDs cannot be used to access another user's private artifact.
- Arbitrary filesystem paths are rejected.
- Public artifact access is limited to explicitly public artifacts.
- Private artifacts require the applicable authorization.
- Returned metadata and content correspond to the same artifact/source reference.

### 9.3 Basic security tests

Verify:

- Required input validation and configured request/file-size limits work.
- Untrusted filenames, metadata, and provider responses are not executed as instructions or commands.
- Public endpoints do not expose private investigation data.
- Private resource ownership/authorization checks work when authentication is enabled.
- Restricted administrative operations cannot be accessed anonymously.
- Approved artifact access rules are enforced on every relevant request.
- Arbitrary user-supplied URLs or filesystem paths cannot be used for unrestricted server-side retrieval.
- CORS configuration uses the intended frontend origins.
- API error messages do not expose stack traces, secrets, database credentials, or local filesystem paths.
- Logs do not contain passwords, access tokens, API keys, or unnecessary raw scientific payloads.

Security tests must follow the requirements in `SECURITY.md`. Do not claim a security control has passed unless the corresponding test actually runs.

### 9.4 External-provider behavior

Core automated tests must use controlled mocks or fixtures for external archive and catalogue services.

Test:

- Successful retrieval.
- Successful empty results.
- Timeout or connection failure.
- Unavailable product.
- Malformed response.
- Corrupt or unreadable retrieved file.
- Provider rate limiting or other documented errors where relevant.

Live-provider tests may be run separately. They must report the actual provider, product/release, date, result, and whether the integration was verified.

A temporary provider outage must not make the core test suite fail merely because a live provider is unavailable.

---

## 10. MVP Release Gates

The MVP is acceptable when all applicable criteria below are true.

### 10.1 Public Explorer

- [ ] Supported coordinate input and validation work.
- [ ] Observation discovery returns valid supported records.
- [ ] Frame manifests use actual observation timestamps.
- [ ] Spectral channels are not represented as separate temporal epochs.
- [ ] Playback and frame navigation use available observations only.
- [ ] Missing frames and coverage gaps are represented honestly.
- [ ] Empty results and provider failures are distinguishable.
- [ ] Public browsing does not start an investigation or agent run.
- [ ] Public artifact access follows the configured access boundary.

### 10.2 Agentic Investigation Engine and User Console

- [ ] The primary investigation-to-report workflow passes end to end on a supported dataset or clearly labelled sample dataset.
- [ ] At least one deterministic scientific method passes its known-answer validation test.
- [ ] Required API and workflow tests pass.
- [ ] PostgreSQL persistence and migration tests pass for the implemented schema.
- [ ] Evidence and available provenance can be retrieved.
- [ ] Reports contain methods, sources, uncertainty where available, and limitations.
- [ ] Frontend loading, success, failure, cancellation, and insufficient-data states behave correctly.
- [ ] Retry and idempotency behavior follows the documented API contract.

### 10.3 Admin Console

- [ ] Implemented administrative routes are protected by backend authorization.
- [ ] Unimplemented administrative features are not presented as functioning controls.
- [ ] Implemented sensitive operational actions are validated and auditable where required.

### 10.4 General release requirements

- [ ] No open critical defect or unresolved release blocker remains.
- [ ] No known scientific-integrity violation remains.
- [ ] Unsupported capabilities are clearly identified.
- [ ] Required setup and test instructions are documented.
- [ ] Known limitations are recorded.
- [ ] Tests not run or blocked are explicitly disclosed rather than marked as passed.

---

## 11. Release Blockers

Do not release if the system:

- Fabricates measurements or presents untraceable results as scientific findings.
- Treats missing data or catalogue no-match as proof of a new object.
- Treats SPHEREx spectral channels as separate time epochs.
- Fabricates temporal frames or observation timestamps.
- Hides a failed quality check.
- Reports a failed, blocked, or cancelled workflow as successful.
- Exposes private investigation resources through public Explorer routes.
- Allows unauthorized access to private artifacts or restricted operations.
- Claims a live archive integration is verified when it has not been demonstrated.
- Cannot complete the required supported Explorer experience or investigation-to-report workflow for the MVP release scope.

---

## 12. Minimal Test Implementation Order

Implement tests in coordination with `IMPLEMENTATION_PLAN.md`. The plan owns feature-development sequencing; the following order describes the test work required as features become available.

1. Set up pytest, PostgreSQL test configuration, and the frontend test runner.
2. Add unit tests for core schemas, input validation, status transitions, and error handling.
3. Create the small synthetic scientific and sky-observation fixture sets.
4. Test PostgreSQL schema migrations, persistence, and repositories.
5. Add Public Explorer API and frontend tests for coordinate queries, observation manifests, actual timestamps, and frame behavior.
6. Add a test confirming that Explorer browsing does not create investigations or agent executions.
7. Test the Explorer-to-Investigation handoff when that capability is implemented.
8. Test the first supported scientific method against a known result.
9. Add API/Workflow Orchestrator integration tests with mocked external services.
10. Add User Console tests for the supported investigation-to-report workflow.
11. Add Admin Console tests only for administrative functionality included in the release.
12. Add the end-to-end and essential security/failure tests.
13. Run the applicable automated test suite and enforce the release gates above.

Do not write specialized tests for scientific features that have not been selected for the MVP. Add them when those capabilities enter the supported scope.

---

## 13. Test Reporting and Evidence

Test results must be reported factually.

Use these result labels:

- `PASS` — the test or command was actually executed and passed.
- `FAIL` — the test or command was executed and failed.
- `NOT RUN` — not executed.
- `BLOCKED` — could not execute because of a named blocker.
- `UNVERIFIED` — external capability or integration has not been demonstrated.
- `PLANNED` — specified but not implemented.

For each substantial test run, record:

- Test group and relevant test IDs.
- Exact command or test suite executed.
- Environment and database used.
- Fixture/version or external source used.
- Result and relevant failure details.
- Known limitations or skipped tests.

A test specification, generated test file, mocked result, or successful HTTP response is not proof that the feature passed acceptance.

Do not report a planned, skipped, blocked, or unexecuted test as successful.

---

**End of document.**
