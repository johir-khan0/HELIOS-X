
# HELIOS-X API Specification

**Document:** `API_SPECIFICATION.md`  

**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  

**Version:** 2.0 (Aligned Draft)

**Status:** Phase 2 specification — Explorer and investigation contracts aligned

**Persistence baseline (defined separately):** PostgreSQL; API contracts remain database-independent  

**Backend framework:** Python, FastAPI, Pydantic  

**API style:** REST over HTTP, JSON request/response bodies  

**Base path:** `/api/v1`

---

## 1. Purpose and Scope

This document defines the API contracts used by the HELIOS-X frontend, backend services, workflow orchestrator, and scientific investigation components. It specifies resource boundaries, endpoints, request and response formats, validation, lifecycle transitions, error handling, pagination, cancellation, report retrieval, versioning, and testing expectations.

The API is a **contract**, not a database schema. Resource representations and payloads must remain independent of the persistence implementation. PostgreSQL, SQLAlchemy, and Alembic details belong in `DATABASE_SCHEMA.md`.

### 1.1 Goals

- Provide a predictable interface for creating and managing scientific investigations.

- Allow the frontend to start long-running workflows and track progress.

- Expose task status, evidence, candidate assessments, and reports with traceable provenance.

- Make validation and errors consistent across endpoints.

- Preserve scientific integrity: results must distinguish measurements, uncertainty, quality outcomes, and hypotheses.

- Make unsupported operations explicit rather than returning fabricated or misleading results.

- Keep the initial implementation small enough for an auditable MVP.

### 1.2 Non-goals

- Defining database tables, indexes, migrations, or SQL queries.

- Requiring every internal component or agent to have its own HTTP endpoint.

- Defining a public third-party API or external SDK in the MVP.

- Treating an unmatched source or candidate as a confirmed planet.

- Replacing deterministic scientific computation with LLM-generated measurements.

- Promising analysis capabilities before the selected dataset and product have passed a feasibility review.

### 1.3 Design principles

1. **Contract-first:** define request and response models before implementing routes.

2. **Explicit lifecycle:** workflow and task state changes are validated.

3. **Traceability:** results link to tasks, data products, methods, versions, and evidence.

4. **Deterministic science:** measurements and numerical analysis come from tested computational code.

5. **Capability-aware:** unsupported requests return a clear status and reason.

6. **Database-independent:** API contracts do not expose table names or ORM objects.

7. **Modular monolith for MVP:** components communicate through typed internal interfaces unless HTTP is justified.

8. **Human review:** candidate prioritization is not equivalent to scientific confirmation.

---

## 2. Architecture and Communication Boundaries

### 2.1 High-level communication

```text

React / TypeScript Frontend

          |

          | HTTPS + JSON: /api/v1/*

          v

FastAPI Application

          |

          +--> Public Explorer services (read-only query and image delivery)
          |       +--> Sky-region query and observation discovery
          |       +--> Frame manifest and controlled artifact delivery
          |       +--> Optional validated image comparisons
          |
          +--> Investigation and resource services

          |

          +--> Workflow Orchestrator

          |       |

          |       +--> Investigation Planner Agent

          |       +--> Data Retrieval Agent

          |       +--> Data Quality Agent

          |       +--> Change Detection Agent

          |       +--> Motion Analysis Agent

          |       +--> Spectral & Photometry Agent

          |       +--> Catalogue Matching Agent

          |       +--> Validation & Hypothesis Agent

          |

          +--> Deterministic Scientific Engine

          +--> Data-source adapters

          +--> Candidate Assessment Service

          +--> Scientific Report Builder

          |

          +--> Persistence layer (specified separately)

```

### 2.2 Communication rules

- The Public Explorer communicates through explicitly designated, read-only, rate-limited Explorer endpoints; it does not need an investigation or agent execution to display observations.
- The authenticated User Console communicates with protected investigation, execution, evidence, candidate, report, and review endpoints.
- The Admin Console uses a separately protected administrative route group; discovering an endpoint path must never grant access.
- The frontend communicates with the backend through versioned HTTP endpoints.

- The FastAPI layer validates input, checks permissions, calls application services, and maps results to public response schemas.

- The Workflow Orchestrator is a deterministic system component, not an AI agent.

- The eight named agents are logical roles. They do not have to run as eight separate processes or expose eight HTTP APIs.

- The Scientific Engine performs numerical and data-processing work through tested code. Agents may request supported operations and interpret structured outputs, but must not invent measurements.

- Data-source adapters encapsulate provider-specific access and normalize outputs.

- Candidate Assessment Service and Scientific Report Builder are system services, not additional agents.

- Internal services should use typed Python interfaces in the modular-monolith MVP. Add internal HTTP boundaries only if deployment needs justify them.

- Long-running archive queries, cutout generation, comparisons, report generation, and scientific investigations should return the appropriate query/job/execution identifier and be tracked through status endpoints rather than holding an HTTP request open.
- Explorer queries and image playback are not investigation executions. Opening, searching, or playing frames must never create an investigation or invoke an agent implicitly.
- An Explorer-to-investigation handoff must be explicit, must reference a server-validated snapshot of selected observations/frames, and must not send image arrays or browser-supplied measurements.

---

## 3. API Conventions

### 3.1 Base URL and versioning

All application routes use the `/api/v1` prefix. The host and port are deployment configuration and must not be hard-coded into clients.

Example local base URL:

```text

http://localhost:8000/api/v1

```

The frontend should use an environment-configured API base URL.

### 3.2 HTTP methods

| Method | Intended use |

|---|---|

| `GET` | Read a resource or collection; must not cause a state-changing operation |

| `POST` | Create a resource or request an operation |

| `PATCH` | Partially update explicitly mutable fields |

| `DELETE` | Delete a resource only where deletion is explicitly supported |

Do not use `GET` to start workflows, cancel work, or trigger report generation.

### 3.3 Content types

- Requests with JSON bodies use `Content-Type: application/json`.

- JSON responses use `Content-Type: application/json`.

- Binary downloads, if introduced, use an appropriate media type such as `application/pdf`.

- Unsupported media types should be rejected consistently.

### 3.4 Identifiers and timestamps

- Public resource IDs are opaque strings; clients must not depend on their internal format.

- Timestamps use ISO 8601 / RFC 3339 with UTC offset, preferably `Z`, for example `2026-10-21T08:30:00Z`.

- Durations are numeric seconds unless a schema explicitly says otherwise.

- Missing and `null` are not interchangeable: omit optional fields when no value is available unless the schema explicitly permits `null`.

### 3.5 Naming and field conventions

- JSON field names use `snake_case`.

- Resource collection names are plural.

- Enum values are stable uppercase strings.

- Unknown fields should be rejected for write requests where practical, so client/server contract mistakes are caught early.

- Responses must not expose database rows, ORM objects, internal filesystem paths, secrets, or unfiltered exception details.

### 3.6 Request correlation

The server should generate a `request_id` for each request and include it in the response envelope or response headers. If the client supplies a valid correlation ID, the server may retain it under a documented header policy. A request ID is for diagnostics and is not an authorization credential.

---

## 4. Resource Model

The initial API uses the following conceptual resources:

| Resource | Purpose | Typical owner |

|---|---|---|

| Explorer Query | Bounded read-only sky-region query and its retrieval outcome | Explorer service |

| Frame Manifest | Versioned, time-ordered set of displayable frame references grouped by compatible sequence | Explorer service |

| Frame | A displayable image plane/product reference associated with an observation, channel/band, and detector array when available | Explorer service / data adapter |

| Artifact | Controlled reference to an original product, cutout, preview, comparison, or report file | Artifact service |

| Image Comparison | Optional validated derived image operation and its diagnostics | Scientific Engine / artifact service |

| Investigation Source Snapshot | Immutable, server-validated set of Explorer observation/frame references and query provenance associated with an investigation/run | Investigation service |

| Target Resolution | Optional authoritative object-name-to-coordinate resolution record, including provenance and ambiguity | Target-resolution adapter/service |

| Investigation | User-defined scientific question and configuration | Application service |

| Execution | One run of an investigation workflow | Workflow Orchestrator |

| Task | A unit of work in an execution | Workflow Orchestrator |

| Data Retrieval | A request and result summary for obtaining source data | Data-source services |

| Observation | Metadata describing a retrieved observation or product | Data adapter / scientific services |

| Measurement | A scientific value with units, uncertainty, method, and provenance | Scientific Engine |

| Evidence | A traceable result or artifact supporting a claim | Evidence service |

| Candidate | An object or source under investigation, not a confirmed discovery | Candidate Assessment Service |

| Candidate Assessment | A versioned assessment of a candidate using documented criteria | Candidate Assessment Service |

| Report | A generated scientific report and its versions | Scientific Report Builder |

| Review | A human review record or review decision | Review service, if enabled |

These are API concepts. Their exact persistence representation and cardinality will be defined in `DATABASE_SCHEMA.md`. An observation is not necessarily one frame: one observation/product may expose multiple spectral planes/channels or detector-array products. A frame is not automatically a separate observation epoch.

---

## 5. Endpoint Inventory

The following inventory defines the initial API surface. Endpoints should be implemented only when their associated capability is available and covered by tests.

### 5.1 System and capability endpoints

| Method | Path | Purpose |

|---|---|---|

| `GET` | `/health` | Liveness check for the API process |

| `GET` | `/ready` | Readiness check for required dependencies |

| `GET` | `/capabilities` | Report deployment-supported Explorer, data-source, product/release, and investigation capabilities with constraints |

`/health` should be lightweight and must not start scientific work. `/ready` may check required local dependencies but must not expose secrets or detailed internal configuration. `/capabilities` is the source of truth for what the current deployment claims to support.

### 5.2 Public Explorer endpoints

These endpoints support the Public SPHEREx Sky Image Explorer independently of the Agentic Scientific Investigation Engine. They are read-only with respect to scientific source data. They may create bounded query/job metadata or derived display artifacts, but they must not create an investigation, candidate, or agent run.

| Method | Path | Priority | Purpose |
|---|---|---|---|
| `POST` | `/explorer/queries` | P0 | Validate a sky-region/FOV request and start bounded observation discovery |
| `GET` | `/explorer/queries/{query_id}` | P0 | Retrieve query status, coverage summary, and retrieval outcome |
| `GET` | `/explorer/queries/{query_id}/observations` | P0 | List observations/products returned for the query |
| `GET` | `/explorer/queries/{query_id}/frames` | P0 | Retrieve a versioned frame manifest grouped into compatible sequences and ordered by actual observation time |
| `GET` | `/explorer/frames/{frame_id}` | P0 | Retrieve normalized frame metadata and controlled artifact references |
| `GET` | `/artifacts/{artifact_id}` | P0 | Retrieve artifact metadata and access state |
| `GET` | `/artifacts/{artifact_id}/content` | P0 | Stream or securely redirect to an authorized image/report artifact |
| `POST` | `/explorer/frames/{frame_id}/cutouts` | Conditional P0 | Request a cutout when it is not already available from the query pipeline |
| `POST` | `/explorer/comparisons` | P1 / conditional | Request a scientifically validated difference/addition/stack/array-combination operation |
| `GET` | `/explorer/comparisons/{comparison_id}` | P1 / conditional | Retrieve comparison operation status and diagnostics |
| `GET` | `/explorer/queries/{query_id}/overlays` | P1 / conditional | Retrieve separately attributed Gaia/WISE or other supported context overlays |
| `POST` | `/target-resolutions` | P1 / optional | Resolve a named target through a documented resolver; return ambiguity explicitly |
| `GET` | `/target-resolutions/{resolution_id}` | P1 / optional | Retrieve target-resolution result and provenance |
| `POST` | `/observations/{observation_id}/spectra` | P1 / conditional | Request an on-demand spectrum when the product and extraction method support it |
| `GET` | `/spectra/{spectrum_id}` | P1 / conditional | Retrieve derived spectral values, wavelength metadata, uncertainty, and provenance |

`POST /explorer/queries` is not an investigation endpoint. Its response must use Explorer-query status/outcome semantics and must not invoke the Workflow Orchestrator's agentic investigation path. The query request and returned artifacts must be subject to configured FOV, time-range, result-count, download-size, concurrency, and rate limits.

A successful query with zero matching observations is a valid result, not a provider failure. The response must report zero records explicitly. A zero-result query is not evidence that no astronomical object exists in the region.

Image blinking, play/pause, previous/next frame, and selection among already-delivered frames are frontend viewer operations. They do not require a separate backend comparison job. Server-side difference, addition, stacking, or detector-array combination endpoints are conditional and must be advertised only when the required product and method have passed validation.

#### 5.2.1 Optional target-name resolution

If named-object lookup is enabled, expose the bounded `POST /target-resolutions` and `GET /target-resolutions/{resolution_id}` operations. It must record resolver/catalogue name and release, query timestamp, returned identifier, resolved coordinates, coordinate frame/epoch, uncertainty/source reference where available, and ambiguity. Ambiguous matches must be returned as ambiguous and must require user selection; do not silently choose a component. Coordinate-based Explorer queries remain available without this optional feature.

### 5.3 Investigations

| Method | Path | Purpose |

|---|---|---|

| `POST` | `/investigations` | Create a direct investigation or an explicit Explorer-origin investigation from a validated source context |

| `GET` | `/investigations` | List investigations |

| `GET` | `/investigations/{investigation_id}` | Retrieve investigation details |

| `PATCH` | `/investigations/{investigation_id}` | Update mutable metadata or configuration |

| `POST` | `/investigations/{investigation_id}/runs` | Request a new workflow execution |

| `GET` | `/investigations/{investigation_id}/runs` | List executions for an investigation |

### 5.4 Executions and tasks

| Method | Path | Purpose |

|---|---|---|

| `GET` | `/executions/{execution_id}` | Retrieve execution status and summary |

| `POST` | `/executions/{execution_id}/cancel` | Request cancellation |

| `GET` | `/executions/{execution_id}/tasks` | List tasks for an execution |

| `GET` | `/tasks/{task_id}` | Retrieve task status and result summary |

| `POST` | `/tasks/{task_id}/retry` | Request retry when retry policy permits |

Task retry is an explicit operation and must not bypass retry limits, validation gates, or dependency checks.

### 5.5 Data retrieval and observations

| Method | Path | Purpose |

|---|---|---|

| `POST` | `/data-retrievals` | Request a supported data retrieval |

| `GET` | `/data-retrievals/{retrieval_id}` | Retrieve status and provenance summary |

| `GET` | `/observations/{observation_id}` | Retrieve normalized observation metadata |

### 5.6 Evidence, measurements, and candidates

| Method | Path | Purpose |

|---|---|---|

| `GET` | `/investigations/{investigation_id}/evidence` | List evidence for an investigation |

| `GET` | `/evidence/{evidence_id}` | Retrieve evidence details |

| `GET` | `/evidence/{evidence_id}/measurements` | List measurements associated with evidence |

| `GET` | `/investigations/{investigation_id}/candidates` | List candidates |

| `GET` | `/candidates/{candidate_id}` | Retrieve candidate details |

| `GET` | `/candidates/{candidate_id}/assessments` | List assessment versions |

Candidate creation and mutation should be performed by the workflow and candidate services unless a reviewed user-facing use case explicitly requires direct editing. An investigation's Explorer source snapshot is immutable; `PATCH /investigations/{investigation_id}` must not rewrite historical query/manifest/frame references or measurements.

### 5.7 Reports and reviews

| Method | Path | Purpose |

|---|---|---|

| `POST` | `/investigations/{investigation_id}/reports` | Request report generation |

| `GET` | `/investigations/{investigation_id}/reports` | List reports |

| `GET` | `/reports/{report_id}` | Retrieve report metadata and content reference |

| `GET` | `/reports/{report_id}/content` | Retrieve report content |

| `GET` | `/reports/{report_id}/versions` | List versions of a report |

| `POST` | `/reports/{report_id}/reviews` | Submit a human review, if review functionality is enabled |

| `GET` | `/reports/{report_id}/reviews` | List reviews, if review functionality is enabled |

The MVP may return Markdown and JSON report representations. PDF export can be added when implemented and tested. A report endpoint must not advertise a format that the deployment cannot generate.

---

### 5.8 Administrative endpoints (conditional)

Administrative routes are not part of the Public Explorer API. Define concrete routes only when an implemented Admin Console operation requires them. Any `/admin/*` route must enforce a backend administrator role, resource-level permission checks, audit logging for consequential changes, and non-disclosure of secrets. Source-registry and capability configuration must not be writable through public capability or Explorer endpoints.

## 6. Request and Response Contracts

### 6.1 Standard response conventions

All JSON success responses MUST use the same envelope convention. Resource and operation responses use `{"data": ... , "request_id": "..."}`. List responses use `data`, `pagination`, and `request_id`. A documented binary artifact-content response is the exception and uses its binary media type plus request/correlation headers. Do not mix enveloped and bare JSON responses within `/api/v1`.

```json

{

  "data": {},

  "request_id": "req_example"

}

```

For list responses:

```json

{

  "data": [],

  "pagination": {

    "limit": 20,

    "offset": 0,

    "total": 0

  },

  "request_id": "req_example"

}

```

For long-running operations, return the created execution or operation resource with its identifier and initial status. Do not wait for the entire workflow to finish before responding.

### 6.2 Create investigation

`POST /api/v1/investigations`

Example request:

```json

{

  "name": "SPHEREx target investigation",

  "scientific_question": "Evaluate whether the available observations support further investigation of a candidate source.",

  "target": {

    "kind": "sky_position",

    "ra_deg": 123.456,

    "dec_deg": -12.345

  },

  "data_request": {

    "source": "spherex",

    "product_ids": [],

    "time_range": null

  },

  "analysis_options": {

    "change_detection": true,

    "motion_analysis": false,

    "spectral_photometry": true,

    "catalogue_matching": true

  }

}

```

Example response (`201 Created`):

```json

{

  "data": {

    "investigation_id": "inv_example",

    "name": "SPHEREx target investigation",

    "status": "CREATED",

    "created_at": "2026-10-21T08:30:00Z"

  },

  "request_id": "req_example"

}

```

The exact target schema must support only target types and search parameters that the selected data adapters can process. The example is illustrative, not a promise that every source/product or analysis option is currently supported.

### 6.3 Start a workflow run

`POST /api/v1/investigations/{investigation_id}/runs`

Request body (send the `Idempotency-Key` HTTP header separately):

```json

{
  "source_snapshot_id": "snapshot_example",
  "requested_analyses": [
    "data_quality",

    "catalogue_matching",

    "spectral_photometry"

  ]

}

```

Example response (`202 Accepted`):

```json

{

  "data": {

    "execution_id": "exec_example",

    "investigation_id": "inv_example",

    "status": "QUEUED",

    "created_at": "2026-10-21T08:31:00Z",

    "status_url": "/api/v1/executions/exec_example"

  },

  "request_id": "req_example"

}

```

For an Explorer-origin investigation, `source_snapshot_id` is required for an Explorer-origin investigation and must identify the immutable, server-validated observation/frame selection used by this execution. The server must verify that the snapshot belongs to the investigation and that its manifest/product references remain valid. For direct investigations, the field may be omitted when no Explorer snapshot is involved. The server must validate that the requested analyses are supported by the selected data products and deployment capabilities. Unsupported work should be rejected before execution where possible, or represented explicitly as `NOT_SUPPORTED` with a reason if discovered during execution.

### 6.4 Retrieve execution status

`GET /api/v1/executions/{execution_id}`

Example response:

```json

{

  "data": {

    "execution_id": "exec_example",

    "investigation_id": "inv_example",

    "status": "RUNNING",

    "progress": {

      "completed_tasks": 2,

      "total_tasks": 6,

      "percent": 33.3

    },

    "started_at": "2026-10-21T08:31:10Z",

    "finished_at": null,

    "task_summary": {

      "QUEUED": 3,

      "RUNNING": 1,

      "SUCCEEDED": 2,

      "FAILED": 0,

      "BLOCKED": 0

    }

  },

  "request_id": "req_example"

}

```

Progress is an estimate derived from task states. If total work is not yet known, `percent` should be `null`, not a fabricated value.

### 6.5 Task response

A task representation should include:

- `task_id`

- `execution_id`

- `agent_role` or `component`

- `task_type`

- `status`

- `attempt`

- `created_at`, `started_at`, `finished_at`

- `depends_on` task identifiers

- `result_summary`, if available

- `error`, if failed

- `quality_outcome`, when applicable

The API must not expose private chain-of-thought or internal hidden reasoning. It may expose concise rationale, structured findings, tool execution summaries, and evidence references suitable for audit.

### 6.6 Evidence and measurement representations

Evidence should expose, where applicable:

- `evidence_id`

- `investigation_id`

- `execution_id` and `task_id`

- `evidence_type`

- `summary`

- `source_references`

- `method_name` and `method_version`

- `created_at`

- `quality_outcome`

- `limitations`

Measurements should expose, where applicable:

- `measurement_id`

- `quantity`

- `value`

- `unit`

- `uncertainty`

- `uncertainty_type`, if known

- `method_version`

- `input_evidence_ids`

- `quality_flags`

- `provenance`

Numeric values must be computed by deterministic, tested scientific code. If a value or uncertainty is unavailable, the API must represent that fact explicitly rather than inventing a number.

### 6.7 Candidate representation

Candidate records should distinguish a candidate from a confirmed discovery. A candidate response may include:

- `candidate_id`

- `investigation_id`

- `designation` or provisional label

- `status`

- `assessment_version`

- `supporting_evidence_ids`

- `contradictory_evidence_ids`

- `alternative_explanations`

- `missing_data`

- `recommended_next_steps`

- `updated_at`

Candidate statuses must use the project’s canonical values:

- `NEEDS_MORE_DATA`

- `NOT_PRIORITIZED`

- `PRIORITIZED_FOR_REVIEW`

- `BLOCKED_BY_QUALITY_ISSUE`

Do not introduce an implicit `CONFIRMED_PLANET` state through the API. Any future confirmed-discovery state would require a separate, explicitly reviewed scientific governance decision.

### 6.8 Report generation and retrieval

`POST /api/v1/investigations/{investigation_id}/reports`

Example request:

```json

{

  "format": "markdown",

  "include_evidence": true,

  "include_methodology": true,

  "include_limitations": true

}

```

Example response (`202 Accepted`):

```json

{

  "data": {

    "report_id": "rep_example",

    "investigation_id": "inv_example",

    "status": "QUEUED",

    "format": "markdown",

    "created_at": "2026-10-21T08:40:00Z"

  },

  "request_id": "req_example"

}

```

`GET /api/v1/reports/{report_id}` returns metadata, status, version, and available formats. `GET /api/v1/reports/{report_id}/content` returns the requested representation when ready. If generation is still in progress, return a documented conflict or accepted-operation response consistently; do not return an empty document as if generation succeeded.

Every report must record the investigation/execution scope, source data references, method versions, limitations, and generation timestamp. Reports should be versioned rather than silently overwritten.

---

### 6.9 Explorer sky-region query

`POST /api/v1/explorer/queries`

The request is a bounded, read-only archive-discovery request. It does not create an investigation or invoke any AI agent.

Example request:

```json
{
  "center": {
    "ra_deg": 133.795129,
    "dec_deg": -7.245147,
    "coordinate_frame": "ICRS",
    "coordinate_epoch": null
  },
  "region": {
    "shape": "RECTANGLE",
    "width_deg": 0.25,
    "height_deg": 0.25
  },
  "observation_time_range": {
    "start": "2025-01-01T00:00:00Z",
    "end": "2026-01-01T00:00:00Z"
  },
  "data_release": null,
  "product_types": [],
  "spectral_channel_ids": [],
  "broad_band_ids": [],
  "detector_array_ids": [],
  "max_observations": 100,
  "include_display_artifacts": true
}
```

The coordinates above are illustrative input values only; this example does not assert archive coverage or a detection at that position. Actual FOV bounds, coordinate frames, time-range limits, allowed releases, result counts, and filter values must come from the active capability configuration and be enforced server-side.

Contract rules:

- `ra_deg` and `dec_deg` must be finite and within the supported celestial-coordinate ranges. The request must state or resolve a supported coordinate frame; `coordinate_epoch` is required when the selected input/reference requires one. Preserve the original user input and the normalized query coordinates.
- `region` must state a supported geometry and explicit angular units. The server enforces configured minimum/maximum FOV and maximum query area. Do not silently clamp an unsupported FOV; return a clear validation/capability result and the accepted bounds.
- Observation time range refers to actual observation/acquisition time, not archive retrieval time, release date, file timestamp, or processing time. If omitted, the server must apply a documented, bounded default window or require the caller to provide one; it must never issue an unbounded archive query. Preserve the original time representation/time scale when the source supplies it and record any normalization applied.
- `spectral_channel_ids`, `broad_band_ids`, and `detector_array_ids` are separate dimensions. Do not infer that there are exactly six usable broad bands or six available array products for every frame, and do not treat spectral channels as separate observation epochs.
- Empty filter arrays mean no filter on that dimension only when the request schema documents that behavior. An omitted release means the configured default release-selection policy, not an arbitrary or latest-looking file.
- The endpoint must use only registered archive adapters and configured query limits. It must not accept arbitrary provider URLs, SQL, filesystem paths, or shell commands.
- A client may send `Idempotency-Key` for safe retry of a submitted query job. The key does not grant access to query results.

Response (`202 Accepted`):

```json
{
  "data": {
    "query_id": "query_example",
    "status": "QUEUED",
    "submitted_at": "2026-11-02T06:00:00Z",
    "status_url": "/api/v1/explorer/queries/query_example",
    "request_summary": {
      "normalized_center": {
        "ra_deg": 133.795129,
        "dec_deg": -7.245147,
        "coordinate_frame": "ICRS"
      },
      "region": {
        "shape": "RECTANGLE",
        "width_deg": 0.25,
        "height_deg": 0.25
      }
    }
  },
  "request_id": "req_example"
}
```

The query may complete quickly or remain queued/running depending on the source adapter. Clients poll the query resource. Do not keep the HTTP request open while archive operations or cutout generation run.

### 6.10 Explorer query status and result outcome

`GET /api/v1/explorer/queries/{query_id}`

The response should include:

- `query_id`, `status`, `submitted_at`, `started_at`, and `finished_at` when available;
- normalized query center, region, filters, and selected release policy;
- `outcome_code` when terminal;
- `observation_count`, `frame_count`, and usable-frame count when known;
- coverage summary and warnings;
- links to observations and frame manifest;
- provider/retrieval attempt summaries without secrets;
- `expires_at` only when the query has a defined expiry.

Explorer query `status` uses the existing canonical task/query execution states: `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRYING`, `CANCELLED`, `BLOCKED`, `NOT_SUPPORTED`, and `INSUFFICIENT_DATA`. Do not create a separate conflicting status enum.

The terminal `outcome_code` distinguishes at least:

- `MULTIPLE_OBSERVATIONS`;
- `SINGLE_OBSERVATION`;
- `NO_MATCHING_OBSERVATIONS`;
- `PARTIAL_RESULTS`;
- `PRODUCT_UNAVAILABLE`;
- `INSUFFICIENT_METADATA`;
- `UNSUPPORTED_QUERY`;
- `INVALID_PRODUCT`;
- `PROVIDER_ERROR`.

These are result outcome codes, not task statuses. In particular, a successful query with no matching observations uses `status: SUCCEEDED`, `outcome_code: NO_MATCHING_OBSERVATIONS`, and zero counts. A provider error uses `FAILED` (after applicable bounded retries) and a provider-related outcome code. A partial result must list missing/unavailable products or frames. A zero-result query must never be presented as evidence that no source exists.

### 6.11 Observation list and frame manifest

`GET /api/v1/explorer/queries/{query_id}/observations`

Returns normalized observation/product records for the query using the standard list envelope and pagination. An observation and a frame are different resources; one observation/product may contain multiple image planes, channels, or detector-array products.

`GET /api/v1/explorer/queries/{query_id}/frames`

Returns a versioned manifest. A manifest should include:

```json
{
  "data": {
    "manifest_id": "manifest_example",
    "query_id": "query_example",
    "manifest_version": "1",
    "generated_at": "2026-11-02T06:01:00Z",
    "frame_count": 1,
    "sequences": [
      {
        "sequence_id": "sequence_channel_example",
        "sequence_dimensions": {
          "spectral_channel_id": "channel_example",
          "broad_band_id": "band_example",
          "detector_array_id": "array_example"
        },
        "ordering": "observation_time_ascending",
        "frames": [
          {
            "frame_id": "frame_example_001",
            "observation_id": "observation_example_001",
            "product_id": "product_example_001",
            "observation_time": "2025-05-01T00:00:00Z",
            "time_scale": "UTC",
            "retrieved_at": "2026-11-02T06:00:30Z",
            "data_release": "release_as_returned_by_archive",
            "processing_version": "processing_version_as_returned",
            "spectral_channel_id": "channel_example",
            "wavelength": {
              "value": 1.0,
              "unit": "micrometre"
            },
            "broad_band_id": "band_example",
            "detector_array_id": "array_example",
            "image": {
              "width_px": 256,
              "height_px": 256,
              "pixel_scale_arcsec": null,
              "unit": "archive_product_unit"
            },
            "wcs": {
              "available": true,
              "metadata_ref": "wcs_example"
            },
            "quality": {
              "calibration_status": "as_returned_by_product",
              "quality_flags": [],
              "contamination_flags": [],
              "mask_artifact_id": null,
              "variance_artifact_id": null,
              "psf_reference": null
            },
            "artifacts": {
              "source_product_artifact_id": "artifact_source_example",
              "cutout_artifact_id": "artifact_cutout_example",
              "preview_artifact_id": "artifact_preview_example"
            }
          }
        ]
      }
    ]
  },
  "pagination": {
    "limit": 20,
    "offset": 0,
    "total": 1
  },
  "request_id": "req_example"
}
```

The values in this schema example are illustrative and must not be persisted as real observations. Optional fields may be omitted when unavailable; if the API uses explicit nulls, the Pydantic schema must permit them consistently. It must not fabricate a wavelength, WCS, quality flag, uncertainty, timestamp, release, or calibration value.

Manifest rules:

- Each frame must link to stable observation and product identities and preserve source/release/processing provenance.
- Sort frames by validated actual observation/acquisition time within a compatible `sequence_id`. Do not sort playback by `retrieved_at`, file timestamp, release timestamp, channel number, or array number.
- Keep spectral channel/wavelength, broad band, detector array, observation epoch, product release, and retrieval time as separate metadata dimensions.
- Multiple channel or array planes belonging to one observation epoch must not be shown as successive time epochs. A timeline sequence should group a consistent channel/band/array/product selection across actual observation times.
- If the selected region has one or zero usable frames, return the truthful manifest/count and outcome; do not generate missing frames to create a video-like sequence.
- The manifest may include multiple sequence groups. The client must allow the user to select a sequence rather than merging incompatible groups into one apparent time series.
- Include frame-quality, contamination, mask, variance, PSF, calibration, and WCS metadata/references where available. Missing values remain unavailable.
- Pagination must not reorder frames between pages. The manifest version and stable ordering key must remain identifiable.

### 6.12 Frame and artifact representations

`GET /api/v1/explorer/frames/{frame_id}` returns normalized frame metadata and the artifact identifiers that the caller is authorized to access. It does not return large image arrays embedded in JSON.

`GET /api/v1/artifacts/{artifact_id}` returns artifact metadata such as:

- artifact ID and type (`SOURCE_PRODUCT`, `CUTOUT`, `PREVIEW`, `COMPARISON`, `REPORT`, or another documented type);
- source observation/product/frame references;
- content media type and byte size when known;
- dimensions and pixel scale for image artifacts when available;
- creation time, checksum when actually computed, product release, processing/method version;
- readiness/access state and a relative content endpoint.

`GET /api/v1/artifacts/{artifact_id}/content` returns a controlled binary stream or an explicitly configured short-lived redirect. It must validate artifact ID, caller/resource access, allowed artifact type, expiry, and size policy. Never return internal filesystem paths, raw storage bucket credentials, arbitrary provider URLs, or unrestricted archive proxy access. HTTP range requests may be supported for large artifacts when tested. A preview or lossy display derivative must not be represented as the authoritative scientific source product.

If an artifact is not ready, return the documented operation/status response or `409 REPORT_NOT_READY`-style resource-not-ready error with a stable, artifact-appropriate error code. Do not return an empty image/document as a successful artifact.

### 6.13 Cutout generation

`POST /api/v1/explorer/frames/{frame_id}/cutouts` is conditional on the archive/product not already supplying a suitable cutout. The request may specify a validated region contained within the frame coverage and supported output dimensions/format. The backend must enforce configured pixel/byte limits and preserve WCS/coordinate metadata where the chosen cutout method supports it.

Return `202 Accepted` with a cutout operation/artifact identifier when generation is asynchronous. The completed artifact must retain a reference to the original product and the exact cutout parameters, method/software version, and timestamp. A request outside the source frame coverage must not be silently clamped.

### 6.14 Optional image comparison operations

`POST /api/v1/explorer/comparisons` is conditional/P1. It accepts stable frame IDs and an explicitly requested operation, such as `DIFFERENCE`, `ADDITION`, `STACK`, or `DETECTOR_ARRAY_COMBINATION`, only if the active capability registry advertises that operation as supported.

The API must validate product/release compatibility, measurement units, channel/wavelength, detector-array identity, WCS, image dimensions/pixel scale, calibration, masks, variance, PSF/background requirements, registration residuals, and operation-specific prerequisites before producing an artifact. If prerequisites fail, return a documented `BLOCKED`, `NOT_SUPPORTED`, or `INSUFFICIENT_DATA` result with diagnostics; do not create a scientifically invalid output.

For `DETECTOR_ARRAY_COMBINATION`, the selected products must meet the validated same-epoch/channel and instrument/product compatibility requirements. It must not combine different observation epochs as a way to claim array combination. Preserve per-array inputs and provenance. Cross-epoch stacking is a separate operation and must not replace original epoch frames; warn that it may smear moving sources or suppress variability.

Blinking is a viewer operation over existing frames, not a scientific subtraction. A visible difference is not by itself a measured motion or variability result. Every derived comparison artifact must identify input frame IDs, operation, method/software version, parameters, quality diagnostics, uncertainty handling, and limitations.

### 6.15 Optional named-target resolution

When enabled, a target-resolution request uses a documented resolver/catalogue and returns resolved object identifier, canonical name/aliases, coordinates, coordinate frame/epoch, positional uncertainty/reference when available, resolver release/version, query time, and alternatives. If multiple plausible components exist (for example, a multi-component system), return an explicit ambiguity response and require user selection. A name resolution is a coordinate cross-reference, not proof that the object has been detected in a SPHEREx image.

### 6.16 Capability response contract

`GET /api/v1/capabilities`

This endpoint is the source of truth for the capabilities actually enabled in the current deployment. It must not advertise planned, unimplemented, unverified, or currently unavailable operations as available. Public responses must omit private source credentials and administrator-only configuration.

The response should describe, as applicable:

- API contract version;
- Public Explorer availability and access class (`PUBLIC_READ_ONLY`);
- supported coordinate frames and coordinate-input rules;
- supported FOV geometries and configuration-backed minimum/maximum dimensions/area;
- maximum time range, observation/frame count, response size, and output dimensions;
- supported archive sources, product families, currently available releases, and processing versions as returned by the adapter/registry;
- availability of observation discovery, frame manifests, image preview/cutout delivery, and optional comparison/overlay/spectral features;
- investigation workflow availability and the analysis operations currently supported by the deployment;
- product-specific constraints and useful reason text for unavailable capabilities.

Capability availability is a capability descriptor, not a task status. Use a documented capability value such as `AVAILABLE`, `EXPERIMENTAL`, `NOT_SUPPORTED`, or `DISABLED`, with optional `reason` and `constraints`. Do not reuse a task execution status to describe deployment capability.

Example response (values are illustrative; actual values must come from runtime configuration and verified adapter checks):

```json
{
  "data": {
    "api_version": "v1",
    "explorer": {
      "availability": "AVAILABLE",
      "access": "PUBLIC_READ_ONLY",
      "coordinate_frames": ["ICRS"],
      "fov": {
        "supported_geometries": ["CIRCLE", "RECTANGLE"],
        "min_width_deg": null,
        "max_width_deg": null,
        "max_area_sq_deg": null
      },
      "limits": {
        "max_observations": 100,
        "max_frames": 200,
        "max_artifact_bytes": null,
        "max_time_span_days": null
      },
      "features": {
        "observation_discovery": {"availability": "AVAILABLE"},
        "frame_manifest": {"availability": "AVAILABLE"},
        "preview_delivery": {"availability": "AVAILABLE"},
        "cutout_generation": {"availability": "EXPERIMENTAL"},
        "image_difference": {"availability": "NOT_SUPPORTED"},
        "detector_array_combination": {"availability": "NOT_SUPPORTED"},
        "catalogue_overlays": {"availability": "DISABLED"},
        "on_demand_spectrum": {"availability": "NOT_SUPPORTED"}
      }
    },
    "data_sources": [],
    "products_and_releases": [],
    "investigation": {
      "availability": "AVAILABLE",
      "supported_analysis_operations": []
    }
  },
  "request_id": "req_example"
}
```

An empty `data_sources`, `products_and_releases`, or analysis list means no corresponding entries are currently advertised; it must not be filled with planned sources. Actual bounds must be returned as configured numeric values when known; `null` means the limit is not supplied in this illustrative schema and must not be interpreted as unlimited. The client must still handle unavailable capabilities and unknown future fields defensively.

### 6.17 Explicit Explorer-to-investigation handoff

The handoff uses the existing `POST /api/v1/investigations` endpoint. No separate `/handoffs` endpoint is required for the MVP. The `source_context` object is optional for direct investigation creation and required when the user creates an investigation from the Public Explorer.

Example Explorer-origin request:

```json
{
  "name": "Investigate selected SPHEREx region",
  "scientific_question": "Evaluate the selected observations for scientifically supported changes.",
  "target": {
    "kind": "sky_region"
  },
  "source_context": {
    "kind": "EXPLORER_SNAPSHOT",
    "explorer_query_id": "query_example",
    "frame_manifest_id": "manifest_example",
    "manifest_version": "1",
    "selected_frame_ids": ["frame_example_001", "frame_example_002"],
    "selected_observation_ids": ["observation_example_001"]
  },
  "analysis_options": {
    "change_detection": true,
    "motion_analysis": false,
    "spectral_photometry": false,
    "catalogue_matching": true
  }
}
```

The client supplies only stable references identifying the selected Explorer results. The server must:

1. verify the query and manifest exist and are in a usable state;
2. verify the manifest version and confirm every selected frame belongs to that manifest/query;
3. resolve observation/product records from trusted server-side data, not from client-supplied descriptions;
4. revalidate selected data releases, product references, actual observation timestamps, channel/band/array dimensions, and required metadata;
5. preserve the normalized sky region, FOV, original and normalized coordinates, query filters, query/manifest identifiers and versions, selected frame/observation/product IDs, archive/release/processing provenance, and relevant quality metadata in an immutable source snapshot;
6. record warnings or reject the handoff if the manifest is stale, required products are missing, references conflict, or the data can no longer be validated.

Do not accept image arrays, base64 FITS/image payloads, browser-calculated measurements, arbitrary artifact URLs, filesystem paths, or client-edited archive metadata as authoritative handoff inputs. If the client also sends target coordinates or region parameters, verify that they agree with the normalized Explorer query; the server-side query record remains authoritative.

Example response (`201 Created`):

```json
{
  "data": {
    "investigation_id": "inv_example",
    "status": "CREATED",
    "origin": "EXPLORER",
    "source_snapshot_id": "snapshot_example",
    "source_context": {
      "explorer_query_id": "query_example",
      "frame_manifest_id": "manifest_example",
      "manifest_version": "1",
      "selected_frame_count": 2,
      "selected_observation_count": 1
    },
    "created_at": "2026-11-02T06:10:00Z"
  },
  "request_id": "req_example"
}
```

Creating the investigation and source snapshot does not start the agentic workflow. The client must make a separate explicit `POST /api/v1/investigations/{investigation_id}/runs` request to start execution. For an Explorer-origin run, that request must include the returned `source_snapshot_id`. A stale or invalid snapshot must return a stable conflict/error such as `409 SOURCE_SNAPSHOT_INVALID`; the server must not silently substitute a new frame, release, or observation set.

### 6.18 Optional on-demand spectral extraction

`POST /api/v1/observations/{observation_id}/spectra` and `GET /api/v1/spectra/{spectrum_id}` are P1/conditional operations. They may be exposed only when the selected product contains suitable spectral information and a validated extraction method is implemented.

A spectrum request must identify the observation/product and any supported channel, aperture/region, spatial extraction, or epoch selection required by the method. A returned spectrum must preserve wavelength values and units, spectral-channel identity, flux/intensity quantity and units, calibration, masks/invalid channels, uncertainty, extraction method/version, input product/artifact references, observation epoch, and limitations. Do not assume 102 usable independent values exist in every product. Spectral channels are wavelength samples, not separate observation times; combining spectra across epochs requires an explicit method and must preserve epoch identity.

### 6.19 Optional catalogue overlay semantics

`GET /api/v1/explorer/queries/{query_id}/overlays` is P1/conditional. Each overlay result must identify its independent catalogue/survey, release/version, query timestamp, coordinate frame/epoch, source identifiers, match/position uncertainty where available, and layer type. Gaia and WISE/NEOWISE overlays remain separate contextual layers; they must not be merged into SPHEREx image pixels or represented as SPHEREx measurements. Overlay presence or absence is not proof of physical identity or source novelty.

## 7. Validation Rules

Validation should occur at multiple layers: request schema, application rules, capability checks, and scientific quality gates.

### 7.1 Schema validation

- Required fields must be present.

- Strings must have documented minimum/maximum lengths.

- Numeric fields must reject non-finite values such as NaN and infinity.

- RA/Dec values, when used, must be within the supported coordinate ranges.

- Enum values must be known and versioned.

- Timestamps must be valid ISO 8601 values.

- Arrays must have documented maximum sizes where resource consumption matters.

- IDs must have the expected syntax and must resolve to an accessible resource.

- Unknown write fields should be rejected where practical.
- Explorer FOV geometry, query time range, spectral-channel, broad-band, detector-array, release, frame-count, pixel-size, and response-size limits must be validated against active capability configuration.
- Frame/artifact IDs in a query manifest or handoff must resolve to the same query/authorized resource scope; client-supplied artifact URLs and filesystem paths are never accepted.

### 7.2 Application validation

- A workflow cannot run if its investigation does not exist or is in an invalid state.

- An execution cannot be cancelled if it is already terminal; respond with a documented state-conflict response or return the unchanged terminal state consistently.

- A task can be retried only if its type supports retry, its retry limit has not been reached, and its dependencies remain valid.

- A report can be generated only for a valid investigation and the requested output format must be supported.

- Data-source and analysis requests must pass the deployment’s capability checks.

- References to evidence, observations, candidates, reports, Explorer queries, manifests, frames, artifacts, and source snapshots must be validated for existence and access scope.
- An Explorer-origin investigation must revalidate its query, manifest version, selected frames, observations, product release, and artifact/source references at handoff time.
- Investigation creation must not implicitly create an execution. A run starts only through the explicit run operation.

### 7.3 Scientific validation

- Quality outcomes use `PASSED`, `FAILED`, `INCONCLUSIVE`, `NOT_AVAILABLE`, or `NOT_APPLICABLE`.

- Evidence gates use `PASSED`, `FAILED`, `UNKNOWN`, or `NOT_APPLICABLE`.

- A quality failure must not be silently converted to success.

- Missing data must not be treated as evidence of a novel object.

- Catalogue no-match is not confirmation of a new object.

- SPHEREx spectral channels must not be treated as separate time epochs.
- Six broad wavelength bands, spectral channels, detector arrays, observation epochs, and retrieval timestamps must remain separate dimensions.
- A successful empty Explorer query must be represented separately from provider failure, unsupported filters, invalid/corrupt products, and insufficient metadata.
- Original scientific products and display previews/cutouts must remain distinguishable.
- Public Explorer query and artifact operations must enforce bounded region, time, result, data-size, and rate limits.
- Optional image comparisons must pass the compatibility/quality checks defined by the scientific methodology before artifact creation.

- Analysis that lacks required data must return `INSUFFICIENT_DATA` or another documented outcome.

- Candidate ranking uses documented, versioned criteria and must not rely only on LLM confidence.

---

## 8. Statuses and State Transitions

### 8.1 Task statuses

Use the canonical task statuses:

- `QUEUED`

- `RUNNING`

- `SUCCEEDED`

- `FAILED`

- `RETRYING`

- `CANCELLED`

- `BLOCKED`

- `NOT_SUPPORTED`

- `INSUFFICIENT_DATA`

### 8.2 Execution statuses

The execution-level status is an aggregate of task and orchestrator state. The initial API may use:

- `QUEUED`

- `RUNNING`

- `SUCCEEDED`

- `FAILED`

- `CANCEL_REQUESTED`

- `CANCELLED`

- `PARTIALLY_COMPLETED`

The implementation must document exact aggregation rules. For example, `SUCCEEDED` should mean the required workflow gates passed, not merely that one task succeeded. `PARTIALLY_COMPLETED` is appropriate only if partial completion is defined and useful to the client.

### 8.3 Quality and evidence outcomes

Quality outcomes:

- `PASSED`

- `FAILED`

- `INCONCLUSIVE`

- `NOT_AVAILABLE`

- `NOT_APPLICABLE`

Evidence gate outcomes:

- `PASSED`

- `FAILED`

- `UNKNOWN`

- `NOT_APPLICABLE`

### 8.4 Allowed transition examples

| From | To | Condition |

|---|---|---|

| `QUEUED` | `RUNNING` | Dependencies and preconditions satisfied |

| `RUNNING` | `SUCCEEDED` | Task completed and required output validated |

| `RUNNING` | `FAILED` | Non-recoverable error or retry limit reached |

| `RUNNING` | `RETRYING` | Retry policy permits another attempt |

| `RETRYING` | `QUEUED` or `RUNNING` | Retry is scheduled or begins |

| Non-terminal status | `CANCELLED` | Cancellation is supported and has completed |

| `QUEUED` | `BLOCKED` | A required dependency or quality gate fails |

| Any applicable status | `NOT_SUPPORTED` | Requested capability is unsupported |

| Any applicable status | `INSUFFICIENT_DATA` | Required input data are unavailable or inadequate |

This table is illustrative; the orchestrator implementation must define the full transition matrix and test it. Terminal states must not be silently reopened. Re-execution should create a new execution or attempt record rather than rewriting historical outcomes.

---

### 8.5 Explorer query and retrieval outcomes

Explorer query/job resources reuse canonical execution-style status values:

- `QUEUED`
- `RUNNING`
- `SUCCEEDED`
- `FAILED`
- `RETRYING`
- `CANCELLED`
- `BLOCKED`
- `NOT_SUPPORTED`
- `INSUFFICIENT_DATA`

A separate `outcome_code` describes the completed retrieval result, including `MULTIPLE_OBSERVATIONS`, `SINGLE_OBSERVATION`, `NO_MATCHING_OBSERVATIONS`, `PARTIAL_RESULTS`, `PRODUCT_UNAVAILABLE`, `INSUFFICIENT_METADATA`, `UNSUPPORTED_QUERY`, `INVALID_PRODUCT`, or `PROVIDER_ERROR`. These outcome codes must not be added to the task status enum.

A successful query with zero records is `SUCCEEDED` with `NO_MATCHING_OBSERVATIONS` and zero counts. A query with valid partial results may be `SUCCEEDED` with `PARTIAL_RESULTS` and explicit missing-frame warnings if the documented query contract has completed. A provider failure after bounded retries is `FAILED`. Unsupported filters/capabilities use `NOT_SUPPORTED`; required-but-inadequate observations/metadata use `INSUFFICIENT_DATA`; a failed prerequisite may use `BLOCKED`.

## 9. Error Response Contract

Use one consistent error envelope for API errors:

```json

{

  "error": {

    "code": "VALIDATION_ERROR",

    "message": "One or more request fields are invalid.",

    "details": [

      {

        "field": "target.ra_deg",

        "issue": "Must be between 0 and 360."

      }

    ]

  },

  "request_id": "req_example"

}

```

### 9.1 Standard HTTP status mapping

| Status | Meaning | Example |

|---|---|---|

| `200 OK` | Successful read or update with a body | Resource retrieved |

| `201 Created` | Resource created | Investigation created |

| `202 Accepted` | Long-running operation accepted | Workflow/report queued |

| `204 No Content` | Success with no response body | Supported deletion |

| `400 Bad Request` | Malformed request or invalid operation | Invalid state transition |

| `401 Unauthorized` | Authentication is missing or invalid | Protected route without valid credentials |

| `403 Forbidden` | Authenticated caller lacks permission | Access to a restricted investigation |

| `404 Not Found` | Resource does not exist or is not visible to caller | Unknown ID |

| `409 Conflict` | Request conflicts with current resource state | Duplicate active run or cancellation conflict |

| `413 Payload Too Large` | Request body exceeds limit | Oversized payload |

| `415 Unsupported Media Type` | Unsupported content type | Non-JSON body on JSON endpoint |

| `422 Unprocessable Content` | Schema or field validation failed | Invalid coordinate |

| `429 Too Many Requests` | Rate limit exceeded | Excessive requests |

| `500 Internal Server Error` | Unexpected server failure | Unhandled application error |

| `503 Service Unavailable` | Required service temporarily unavailable | Data provider or dependency unavailable |

Use `422` for schema validation and reserve `400` for malformed requests or application-level invalid operations, according to the final FastAPI exception mapping. Do not leak stack traces or secret values in public error messages.

### 9.2 Stable error codes

At minimum, define and test these codes:

- `VALIDATION_ERROR`

- `RESOURCE_NOT_FOUND`

- `INVALID_STATE_TRANSITION`

- `CAPABILITY_NOT_SUPPORTED`

- `INSUFFICIENT_DATA`

- `QUALITY_GATE_FAILED`

- `IDEMPOTENCY_CONFLICT`

- `CANCELLATION_NOT_POSSIBLE`

- `RETRY_NOT_ALLOWED`

- `REPORT_NOT_READY`

- `RATE_LIMIT_EXCEEDED`

- `INTERNAL_ERROR`

- `DEPENDENCY_UNAVAILABLE`
- `EXPLORER_QUERY_NOT_FOUND`
- `EXPLORER_FRAME_NOT_FOUND`
- `ARTIFACT_NOT_FOUND`
- `ARTIFACT_NOT_READY`
- `INVALID_FOV`
- `QUERY_LIMIT_EXCEEDED`
- `FRAME_MANIFEST_STALE`
- `SOURCE_SNAPSHOT_INVALID`
- `AMBIGUOUS_TARGET`
- `COMPARISON_NOT_SUPPORTED`

The HTTP status and error code must be consistent across endpoints. `NO_MATCHING_OBSERVATIONS` is a successful query outcome, not an API error code. `ARTIFACT_NOT_READY` and `REPORT_NOT_READY` must not return an empty successful content body.

---

## 10. Pagination, Filtering, and Sorting

### 10.1 Pagination

List endpoints should use `limit` and `offset` in the MVP.

Example:

```text

GET /api/v1/investigations?limit=20&offset=0

```

Recommended initial rules:

- Default `limit`: 20

- Maximum `limit`: 100

- `offset`: integer greater than or equal to 0

- Return the actual `limit`, `offset`, and `total` in response metadata.

- Apply stable ordering to avoid unpredictable list results.

- Do not paginate single-resource endpoints.

Example response:

```json

{

  "data": [],

  "pagination": {

    "limit": 20,

    "offset": 0,

    "total": 0

  },

  "request_id": "req_example"

}

```

### 10.2 Filtering and sorting

Only documented fields may be filtered or sorted. Proposed filters include:

- Investigations: `status`, `created_after`, `created_before`

- Executions: `status`, `investigation_id`, `created_after`, `created_before`

- Tasks: `status`, `agent_role`, `task_type`

- Candidates: `status`, `investigation_id`

- Reports: `status`, `format`, `investigation_id`

Sort syntax may use `sort_by=created_at&sort_order=desc`. Invalid fields or directions should return a validation error. Filters must not allow arbitrary database query expressions.

---

## 11. Long-running Operations, Cancellation, Retries, and Idempotency

### 11.1 Asynchronous execution

Workflow runs and report generation should be asynchronous:

1. Client submits a request.

2. Server validates the request and capability requirements.

3. Server creates an execution/operation record.

4. Server returns `202 Accepted` and a resource ID.

5. Client polls the status endpoint.

6. Client retrieves results or report content when ready.

The API must not keep a normal HTTP request open for the full duration of a scientific workflow.

### 11.2 Cancellation

`POST /api/v1/executions/{execution_id}/cancel`

Example request:

```json

{

  "reason": "User requested cancellation."

}

```

Example response:

```json

{

  "data": {

    "execution_id": "exec_example",

    "status": "CANCEL_REQUESTED",

    "message": "Cancellation has been requested; task shutdown is not yet confirmed."

  },

  "request_id": "req_example"

}

```

Rules:

- Cancellation is a request, not a guarantee of immediate termination.

- The orchestrator stops scheduling new tasks after cancellation is accepted.

- Active tasks are cancelled where their underlying operations support cancellation.

- If a task cannot be interrupted safely, the system records that fact and waits for a safe boundary where possible.

- The execution reaches `CANCELLED` only after the orchestrator confirms the defined cancellation completion criteria.

- Repeated cancellation requests should be idempotent or return a documented conflict.

- Terminal executions are not rewritten to `CANCELLED`.

### 11.3 Retry

- Retry must follow a documented policy and maximum attempt count.

- Only retryable task types and failure classes may be retried.

- Retry must preserve earlier attempt records and outputs.

- A retry must not duplicate evidence or reports without explicit versioning/idempotency behavior.

- Manual retry requests must be validated by the orchestrator.

### 11.4 Idempotency

For create/run/report-generation/Explorer-query operations that may be retried by clients, use the `Idempotency-Key` HTTP header as the single API-wide mechanism. Do not also accept a body-level idempotency field.

- The same key and same effective request should return the same operation/resource result.

- The same key with a different effective request should return `409 Conflict` with `IDEMPOTENCY_CONFLICT`.

- Key scope and retention period must be documented.

- Idempotency must be implemented at the application/service boundary; it must not depend on client-side behavior alone.
- Apply the same documented key-scope and retention policy to Explorer query creation, investigation creation, run creation, cutout/comparison jobs, and report-generation operations where retry duplication would be harmful.

---

## 12. Security and Access Control

Security requirements depend on deployment context, but the product access boundary is normative: the Public Explorer exposes only explicitly allowlisted read-only operations, while investigations and administration remain protected resources.

| Route/resource class | Public visitor | Authenticated user | Reviewer | Administrator |
|---|---:|---:|---:|---:|
| Public `/capabilities` fields and `/explorer/*` read/query routes | Allow, rate-limited | Allow | Allow | Allow |
| Public-designated image/frame artifacts | Allow through controlled delivery | Allow | Allow | Allow |
| Private investigations, runs, tasks, evidence, candidates, reports, source snapshots | Deny | Own/authorized | Assigned/authorized | Only with explicit audited permission |
| Human review submission | Deny | Only with reviewer permission | Assigned resources | Only with explicit reviewer/admin policy |
| `/admin/*` configuration/operations | Deny | Deny | Deny | Allow by explicit role and audit policy |

The table is the API access boundary, not evidence that authentication or role-management infrastructure has already been implemented. Exact authentication, ownership and role semantics are owned by `SECURITY.md`.

### 12.1 MVP local development

- Bind the development server to localhost unless remote access is explicitly needed.

- Keep credentials, tokens, and provider secrets in environment configuration, never in source control.

- Configure CORS with explicit allowed origins rather than unrestricted wildcard access for credentialed requests.

- Set request body and upload limits.

- Validate all user-controlled IDs, filters, paths, and query parameters.

- Never allow arbitrary filesystem paths or shell commands through API inputs.
- Publicly accessible Explorer routes are limited to query/discovery, public capability metadata, frame metadata/manifests designated for public access, and controlled artifact delivery.
- Apply strict server-configured limits to RA/Dec, FOV area, time span, product/release filters, spectral channels, detector arrays, frame count, output dimensions, artifact size, query duration, concurrency, and request rate.
- Validate product and artifact identifiers against registered sources and the controlled artifact store; never expose an unrestricted IRSA/provider proxy or user-supplied remote URL.
- Treat query IDs as references, not authorization tokens. Apply short-lived signed delivery URLs only where the security implementation defines them; never persist them as scientific provenance.

- Do not expose raw FITS files or local files unless an authorized, controlled download mechanism exists.

- Redact secrets and sensitive payloads from logs.

### 12.2 Before public deployment

- Add and test authentication and authorization.

- Define ownership/access policy for investigations, evidence, and reports.

- Apply rate limits and request-size limits.

- Use HTTPS through the deployment environment.

- Define retention, backup, and deletion rules.
- Require authentication and resource-level ownership/role checks for private investigations, executions, source snapshots, evidence, candidates, reports, reviews, cancellation, and retry operations. Public Explorer query visibility must not make an associated private investigation or its artifacts public.
- Require strong authentication, administrator roles, and audit records for `/admin/*` routes.

- Review CORS, security headers, dependency vulnerabilities, and secret rotation.

- Restrict operational endpoints so health checks do not disclose internal details.

The API specification does not mandate a specific identity provider for the MVP. The choice should be made before the system is exposed to untrusted users.

---

## 13. Observability and Auditability

Each request and long-running operation should be traceable.

- Include a request/correlation ID in responses and structured logs.

- Log route, method, response status, duration, and relevant resource IDs.

- Record execution ID, task ID, attempt number, component/agent role, and method version for workflow events.

- Store scientific provenance in structured evidence/results rather than relying only on text logs.

- Avoid logging secrets, access tokens, or unnecessary personal data.

- Expose concise error summaries to clients and retain detailed diagnostics in protected logs.

- Keep timestamps and state changes auditable.

- Report generation must record source references, method versions, limitations, and report version.
- Explorer query logs must retain query ID, normalized region/FOV, filter/release summary, provider attempt IDs, outcome, record counts, latency, and request ID without logging secrets or unnecessary personal data.
- Frame manifests and artifacts must retain their manifest/artifact version, source product identity, release/processing version, and checksum only when computed.
- Handoff audit records must link the Explorer query/manifest version, selected frame and observation IDs, created source snapshot ID, investigation ID, and subsequent execution IDs.

Observability tools such as OpenTelemetry or Sentry may be introduced when justified; they are not mandatory dependencies for the first local MVP.

---

## 14. API Versioning and Compatibility

- Use `/api/v1` for the first stable contract.

- Adding optional response fields is generally backward-compatible if clients ignore unknown response fields.

- Adding a required request field, changing a field's meaning/type, removing an endpoint, or changing status semantics is breaking.

- Breaking changes require a new API version or a documented migration plan.

- Enum additions can break clients that assume a closed set; clients should handle unknown statuses safely.

- Keep OpenAPI output under version control or generate it in CI and compare changes.
- Public Explorer share URLs should encode only the minimal reproducible viewer state (normalized RA/Dec, coordinate frame where needed, FOV geometry/units, time filter, and selected channel/band/array when supported). The frontend route owns the exact URL parameter spelling; the API must return normalized values so shared links can be reconstructed.
- Share URLs must not contain private investigation IDs, access tokens, signed artifact URLs, local paths, or administrative state. Reopening a URL must revalidate every parameter against current capability limits.

- Document deprecated fields and a removal window before removing them from a stable version.

- Database migrations must not silently change the public API contract.

---

## 15. OpenAPI and Implementation Guidance

FastAPI and Pydantic should be the source of truth for executable request/response validation. This document defines the intended contract; route models and generated OpenAPI must remain aligned with it.

Recommended implementation structure:

```text

backend/src/helios_x/

├── api/

│   ├── main.py

│   ├── dependencies.py

│   ├── errors.py

│   └── routes/

│       ├── health.py

│       ├── capabilities.py
│       ├── explorer_queries.py
│       ├── explorer_frames.py
│       ├── artifacts.py
│       ├── explorer_comparisons.py  # conditional/P1
│       ├── target_resolutions.py    # optional
│       ├── investigations.py

│       ├── executions.py

│       ├── tasks.py

│       ├── data_retrievals.py

│       ├── observations.py

│       ├── evidence.py

│       ├── candidates.py

│       └── reports.py

├── schemas/

│   ├── common.py
│   ├── explorer.py
│   ├── artifact.py

│   ├── investigation.py

│   ├── execution.py

│   ├── task.py

│   ├── data_retrieval.py

│   ├── evidence.py

│   ├── candidate.py

│   └── report.py

└── services/

    └── ...

```

This is a proposed organization; align it with the final repository structure before implementation.

Implementation rules:

- Keep route handlers thin.

- Put business rules in application services and lifecycle logic in the orchestrator.

- Use Pydantic models for request and response validation.

- Avoid returning ORM models directly.

- Use dependency injection for services and configuration.

- Keep scientific computations outside route handlers.

- Generate OpenAPI and inspect it in tests.

- Keep internal exception details out of public responses.

- Avoid database-specific assumptions in API schemas.

---

## 16. Testing and Acceptance Criteria

### 16.1 Contract tests

- Every endpoint has a request/response schema and documented status codes.

- Invalid payloads return the documented error envelope.

- Unknown IDs return a consistent not-found response.

- Pagination limits and filters are validated.

- API responses never expose secrets, stack traces, or internal filesystem paths.

- Generated OpenAPI contains the implemented routes and schemas.

### 16.2 Public Explorer acceptance tests

The public Explorer contract must demonstrate:

- Coordinate and FOV validation against configured bounds, including RA wraparound and supported coordinate-frame handling.
- A query that returns multiple observations, one observation, zero observations, and a provider failure produces distinct status/outcome semantics.
- Frame manifests are sorted by actual observation time within each compatible sequence.
- Channel, broad-band, detector-array, observation-epoch, release, and retrieval-time dimensions remain distinct.
- Multiple spectral planes/arrays from one observation are not represented as multiple temporal epochs.
- Frame metadata preserves source/product/release/processing references, WCS/units/quality metadata, and truthful unavailable fields.
- Preview/cutout artifacts are delivered through controlled artifact routes; original products and lossy display derivatives remain distinguishable.
- Public endpoints enforce FOV, time-span, frame-count, file-size, concurrency, timeout, and rate limits.
- A viewer operation does not create an investigation or agent run.
- Optional comparisons are rejected/blocked when compatibility or mandatory quality checks fail.
- A shareable Explorer state contains no private investigation IDs, secrets, local paths, or signed artifact credentials.

### 16.3 Investigation workflow acceptance test

A supported MVP workflow must demonstrate:

1. Create an investigation.

2. Retrieve the investigation.

3. Start a run and receive `202 Accepted` with an execution ID.

4. Poll execution status and observe valid state transitions.

5. Retrieve task status and structured result summaries.

6. Retrieve evidence with provenance and quality outcomes.

7. Retrieve candidate assessments if the selected workflow produces candidates.

8. Request a report and retrieve its content after completion.

9. Confirm the report describes uncertainty, limitations, methods, and source references.

10. Confirm that unsupported or insufficient-data analysis is represented explicitly and does not fabricate results.

### 16.4 Cancellation and retry tests

- Cancellation of a queued execution.

- Cancellation of a running execution with cancellable tasks.

- Cancellation where a task cannot stop immediately.

- Repeated cancellation requests.

- Cancellation of an already terminal execution.

- Retry of a retryable task.

- Rejection of retry for a non-retryable or exhausted task.

- Preservation of prior attempt history.

### 16.5 Scientific integrity tests

- No catalogue match is not represented as a confirmed new object.

- Missing data is not interpreted as a detection.

- SPHEREx spectral channels are not treated as separate time epochs.

- Measurements include method/provenance metadata when available.

- Failed quality gates remain visible.

- Candidate prioritization uses documented criteria and does not rely only on LLM confidence.

### 16.6 Release gate

The API is ready for MVP integration when:

- The primary investigation-to-report flow works end to end on a supported dataset or clearly labelled sample dataset.

- All required routes have schemas and automated tests.

- State transitions, errors, pagination, and cancellation behavior are tested.

- OpenAPI documentation matches the implementation.

- API resources are independent of database table names.

- No endpoint claims support for an unimplemented data source or scientific operation.
- The Public Explorer can complete the coordinate/FOV-to-observation/frame-manifest-to-artifact journey independently of the investigation workflow.
- The Explorer-to-investigation handoff persists an immutable, server-validated source snapshot, and a separate explicit run request is required to execute analysis.

---

## 17. Decisions to Finalize Before Implementation

The following choices remain implementation decisions and must be recorded before they affect clients:

1. Whether authentication is needed for local MVP use or only before public deployment.

2. Whether status polling alone is sufficient, or server-sent events are needed for progress updates.

3. Which SPHEREx product(s) and analysis capabilities pass feasibility review.

4. Whether report content is returned inline or through a controlled download route for each format.

5. The exact execution-level status aggregation rules.

6. The idempotency-key header format and retention window.

7. Which review actions and user roles are included in the MVP.

8. The final list of filterable/sortable fields.

9. Whether data retrieval is exposed as a standalone resource or only as a workflow task.

10. Confirm through OpenAPI and contract tests that the standard JSON envelope defined in Section 6.1 is implemented consistently.
11. Exact configured FOV/time-range/frame-count limits and query-job polling interval/retention.
12. Exact artifact streaming/redirect policy, signed URL lifetime if used, and range-request support.
13. The frame-manifest paging/sequence grouping contract and manifest retention/expiry.
14. Whether optional target-name resolution, image comparisons, spectral extraction, and Gaia/WISE overlays are enabled in the MVP or deferred.

These decisions should be resolved only when needed by the MVP; do not overbuild optional features before the end-to-end workflow is working.

---

## 18. Cross-Document Contract Ownership and Alignment

This API specification owns exact HTTP methods, paths, request/response payloads, status/error mappings, pagination, idempotency, and public endpoint access contracts. It must remain aligned with the following documents:

| Document | Contract ownership relevant to this API |
|---|---|
| `PRD.md` | Product identity, Explorer and Agentic Engine journeys, priorities, and acceptance intent |
| `SYSTEM_ARCHITECTURE.md` | Product surfaces, component boundaries, API groups, execution and access architecture |
| `BACKEND.md` | FastAPI route/service implementation boundaries, artifact delivery, and internal module placement |
| `DATA_SOURCE_SPEC.md` | Archive query semantics, release/product metadata, observation/frame meaning, and source limitations |
| `SCIENTIFIC_METHODOLOGY.md` | Scientific prerequisites, compatibility, registration, uncertainty, image comparison, and interpretation rules |
| `INVESTIGATION_WORKFLOW.md` | Explicit handoff, immutable source snapshot, task order, quality/evidence gates, and run lifecycle |
| `AGENT_SPECIFICATIONS.md` | Eight logical agent roles, tool boundaries, output contracts, and failure behavior |
| `DATABASE_SCHEMA.md` | Persistence fields, relations, constraints, migrations, and retained history |
| `SECURITY.md` | Authentication, authorization, public/private/admin access policies, artifact controls, and audit rules |
| `FRONTEND.md` / `UI_UX_SPECIFICATION.md` | Screen behavior, viewer URL state, polling, and presentation of API status/outcome states |

### 18.1 Non-negotiable API boundaries

1. Public Explorer query, manifest, and artifact routes must operate without creating an investigation or invoking an agent.
2. Only an explicit `POST /investigations` handoff with validated Explorer references may create an Explorer-origin investigation.
3. A source snapshot is immutable for each investigation/execution scope. The client supplies references to select records; the server resolves and records authoritative source metadata.
4. `POST /investigations/{id}/runs` is the explicit workflow-start operation. Investigation creation alone does not start an agentic run.
5. Frame sequence ordering uses actual observation/acquisition time, not retrieval time, release timestamp, file timestamp, spectral-channel number, or detector-array number.
6. Spectral channels, broad wavelength bands, detector arrays, observation epochs, product releases, and retrieval timestamps remain distinct fields.
7. A frame manifest is a viewer data structure, not scientific proof of variability or motion. Scientific measurement requires the gates and deterministic methods defined by the scientific documents.
8. Display previews and cutouts are not automatically authoritative scientific input files. Artifact metadata must preserve their relationship to the original source product.
9. Optional comparison, spectral extraction, detector-array combination, and catalogue-overlay APIs must be capability-gated and must not be advertised as available until implemented and validated.
10. No API response, candidate status, score, or report may claim automated confirmation of Planet X or another new planet.

### 18.2 Implementation status and open decisions

This document defines the intended API contract. It does not claim that any route, archive adapter, artifact pipeline, authentication provider, comparison algorithm, or NASA target test has already been implemented. Before release, resolve the configured FOV/query bounds, actual archive access route, release/product constraints, query/job expiry, artifact delivery mechanism, and supported optional operations through the owning documents and implementation tests.

## Appendix A. Suggested Initial Schema Names

These Pydantic schema names are suggested starting points, not final database models:

- `ExplorerSkyQueryRequest`
- `ExplorerQueryResponse`
- `ExplorerObservationResponse`
- `FrameManifestResponse`
- `FrameResponse`
- `ArtifactResponse`
- `ArtifactContentResponse`
- `CutoutCreateRequest`
- `ImageComparisonCreateRequest`
- `TargetResolutionRequest` (optional)
- `InvestigationSourceContext`
- `InvestigationSourceSnapshotResponse`
- `InvestigationCreateRequest`

- `InvestigationUpdateRequest`

- `InvestigationResponse`

- `InvestigationListResponse`

- `ExecutionCreateRequest`

- `ExecutionResponse`

- `ExecutionCancelRequest`

- `TaskResponse`

- `TaskRetryRequest`

- `DataRetrievalCreateRequest`

- `DataRetrievalResponse`

- `ObservationResponse`

- `EvidenceResponse`

- `MeasurementResponse`

- `CandidateResponse`

- `CandidateAssessmentResponse`

- `ReportCreateRequest`

- `ReportResponse`

- `ReportContentResponse`

- `ReviewCreateRequest`

- `ApiErrorResponse`

- `PaginationMetadata`

## Appendix B. Canonical Project Status Values

### Task statuses

`QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRYING`, `CANCELLED`, `BLOCKED`, `NOT_SUPPORTED`, `INSUFFICIENT_DATA`

### Quality outcomes

`PASSED`, `FAILED`, `INCONCLUSIVE`, `NOT_AVAILABLE`, `NOT_APPLICABLE`

### Evidence gate outcomes

`PASSED`, `FAILED`, `UNKNOWN`, `NOT_APPLICABLE`

### Candidate statuses

`NEEDS_MORE_DATA`, `NOT_PRIORITIZED`, `PRIORITIZED_FOR_REVIEW`, `BLOCKED_BY_QUALITY_ISSUE`

---

## Appendix C. Explorer and NASA Target API Test Matrix

The following scenarios validate API behavior and honest data reporting. They do not require a positive detection or guarantee that every target has multiple usable observations.

| Target | API test focus | Required response behavior |
|---|---|---|
| Barnard's Star | Coordinate/name resolution where enabled; query; frame manifest | Use a recorded authoritative target reference; report actual coverage and one/zero/multiple frames honestly |
| Epsilon Indi B | Component disambiguation; query; manifest | Preserve ambiguity and selected component reference; do not silently collapse components |
| WISE 0855 | Query using the challenge-supplied coordinates; query/manifest correctness | Preserve supplied coordinates and coordinate conventions; report actual archive coverage without promising a detection |
| RW Aurigae | Target resolution where enabled; time-domain query | Report actual observation timestamps and coverage; do not infer variability from playback alone |
| R Draconis | Target resolution where enabled; time-domain query | Report actual observation timestamps and coverage; do not infer variability from playback alone |

For each test, verify query parameters, resolver provenance when used, product/release availability, status/outcome semantics, frame ordering, sequence grouping, missing metadata behavior, and request/artifact provenance. A valid zero-result or one-frame outcome may pass when it is the truthful response for the archive state under test.

**End of document.**
