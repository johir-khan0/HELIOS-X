# HELIOS-X — Database Schema Specification



**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  

**Status:** Canonical persistence specification for the MVP — PostgreSQL selected  

**Baseline:** PostgreSQL + SQLAlchemy 2.x + Alembic; optional Redis cache; external controlled artifact storage  

**Version:** 2.0  

**Purpose:** Define logical entities, fields, relationships, integrity rules, provenance, versioning, migrations, and tests.



> This document defines the intended design; it does not claim that the database has been implemented or tested. PostgreSQL is the final selected primary relational database for HELIOS-X. This decision supersedes earlier SQLite references in draft documents; those documents must be synchronized before implementation. Before coding, reconcile it with `PRD.md`, `SYSTEM_ARCHITECTURE.md`, `SCIENTIFIC_METHODOLOGY.md`, `DATA_SOURCE_SPEC.md`, `INVESTIGATION_WORKFLOW.md`, `API_SPECIFICATION.md`, `SECURITY.md`, `OPERATIONS.md`, and `BACKEND.md`. If contracts conflict, record and resolve the conflict at the authoritative source instead of silently creating a competing contract.



## 1. Goals and non-negotiable principles



The database is the durable record of what was requested, what ran, which source data were retrieved, which checks were applied, what deterministic results were produced, how each result was derived, which candidate assessments were made, and which report/review versions followed.



- Preserve provenance from every quantitative report claim to source data, method, parameters, software/data versions, and evidence.

- Separate source observations, deterministic measurements, quality outcomes, evidence, agent interpretations, candidate assessments, and human reviews.

- Preserve historical executions and results. Re-analysis must not silently overwrite earlier results.

- Represent failure, cancellation, blocked work, unsupported operations, insufficient data, and successful empty results distinctly.

- Use PostgreSQL as the single durable relational system of record. Keep the backend as a modular monolith for the MVP; do not add distributed infrastructure without a demonstrated need.

- Keep large FITS/image/spectral files outside relational rows in controlled artifact storage. Persist metadata, identifiers, checksums when computed, and controlled artifact references.

- Never store provider credentials or secrets in database records.

- Keep ORM schema separate from Pydantic/API contracts. Public APIs must not expose table names or ORM objects directly.



## 2. Database conventions and selected platform

### 2.1 Final database decision

- **Primary relational database: PostgreSQL.** Use PostgreSQL consistently in local development, CI/integration tests, staging, and production. SQLite is not the application database or an automatic fallback.
- **ORM and migrations:** SQLAlchemy 2.x typed declarative models and Alembic.
- **Driver:** use a maintained PostgreSQL driver through the SQLAlchemy PostgreSQL dialect (for example, `psycopg`). Pin the selected driver/version in the backend dependency lockfile.
- **Cache:** Redis is optional and is not the durable system of record. Introduce it only for a measured or explicitly required cache, rate-limit, temporary-state, or queue-coordination use case. Loss of Redis must not erase investigations, executions, tasks, evidence, measurements, or reports.
- **Large scientific files:** use a controlled artifact-storage adapter backed initially by the approved deployment storage (local private storage or object storage). Do not store FITS files, full image arrays, large spectra, or report binaries in PostgreSQL rows.
- **No mandatory PostGIS dependency in the first migration.** Coordinate filtering may initially use validated adapter queries and indexed numeric bounds. Add a spatial extension only after a specific query pattern and operational requirement are demonstrated and tested.

The database choice is final at the product/specification level. The exact PostgreSQL server version, hosting model, pool settings, and backup service are deployment decisions that must be pinned in `OPERATIONS.md` before deployment; they do not reopen the SQLite-versus-PostgreSQL choice.

### 2.2 PostgreSQL conventions

- Use PostgreSQL native `UUID` for application-generated identifiers. Generate IDs in the application unless a specific database-generated strategy is documented.
- Use `TIMESTAMPTZ` for instants. Store UTC instants; serialize as ISO 8601/RFC 3339 with an explicit UTC offset in API responses.
- Use `JSONB` for validated, bounded, versioned parameters and source-specific metadata. JSONB does not replace relational foreign keys for core entities, and JSON payloads must have size/depth limits.
- Use `NUMERIC(p, s)` only where exact decimal semantics are needed. Use `DOUBLE PRECISION` for scientific floating-point quantities when appropriate to the algorithm and source representation. Document units, precision, and conversion policy per measurement family; do not imply precision beyond the input/method.
- Use `CHECK`, `UNIQUE`, `NOT NULL`, and foreign-key constraints for invariants that can be enforced in PostgreSQL. Keep lifecycle transition validation in the orchestrator/service layer as well.
- Use `ON DELETE RESTRICT` for scientific-history relationships by default. Prefer archival/retention workflows over cascading deletion.
- Use short transactions and explicit transaction boundaries. Never hold a database transaction open during archive calls, object-store transfers, or LLM calls.
- Use row-level locking or another proven transactional strategy when allocating monotonic run numbers or claiming idempotency keys under concurrent requests.
- Add B-tree indexes for measured access patterns. Add partial or GIN indexes only when supported by actual query plans and representative load tests; do not create blanket indexes on every JSONB field.
- Keep database URL, usernames, passwords, TLS settings, and secret references in deployment configuration/secret management, never in source control or persisted business records.
- Configure connection pooling, statement/lock timeouts, and health checks through deployment configuration. Pool size must be coordinated with the total application worker count.
- Scientific result records should be append-only or versioned. Corrections create a new version or auditable correction, not a silent rewrite.

### 2.3 Runtime configuration

The backend must read its SQLAlchemy connection URL from environment/secret configuration. Example shape only:

```text
DATABASE_URL=postgresql+psycopg://<user>:<password>@<host>:5432/<database>
```

Do not commit real credentials. Use a separate least-privilege runtime role and migration role. The runtime role should not have schema-owner or unrestricted DDL privileges. Use TLS when the database is reached over an untrusted network.

### 2.4 Transaction and concurrency rules

- Create an execution and its initial task graph in one short transaction.
- Persist a task attempt/result/evidence relationship atomically where the operation contract requires it.
- Persist a measurement and its provenance/evidence links atomically.
- Persist a candidate assessment and its criteria/gate results atomically.
- Persist a report and its claims/evidence links atomically.
- Persist an Explorer-origin investigation and its immutable source snapshot atomically.
- Use unique constraints plus transactional handling for idempotency; client-side request guards are not sufficient.
- Never hold a transaction open while computing images, retrieving archive products, generating cutouts, or invoking models.


## 3. Entity map

```text
explorer_queries
  ├── data_retrievals ── observations ── observation_frames ── artifacts
  └── frame_manifests ── frame_manifest_items ── observation_frames

investigations
  ├── investigation_source_snapshots ── snapshot_items ── observations/frames
  ├── executions ── tasks ── task_attempts
  │    ├── task_dependencies
  │    ├── data_retrievals
  │    ├── measurements
  │    ├── quality_checks
  │    ├── evidence_records ── evidence_links
  │    ├── candidates ── candidate_assessments
  │    ├── catalogue_matches / target_resolutions
  │    ├── agent_runs
  │    ├── image_comparisons / derived_spectra (conditional)
  │    └── reports ── report_claims ── report_claim_evidence
  └── human_reviews

idempotency_records
artifacts (referenced by observations, frames, comparisons, spectra, and reports)
external_overlay_queries (optional)
```

This is the target logical model, not a requirement to implement every optional table in the first migration. The P0 Explorer path needs bounded query state, normalized observations, frame records/manifests, and controlled artifact metadata. The P0 investigation path needs immutable source snapshots, executions, tasks, measurements, evidence, and report provenance. Optional comparison, named-target resolution, external overlay, and derived-spectrum entities are conditional on the corresponding capability passing feasibility and validation. Do not create unused tables/modules simply to make the repository look complete.

An Explorer query is independent of an investigation. Browsing, frame playback, or opening an image must not create an investigation, execution, agent run, candidate, or human review. An investigation created from Explorer selections references a server-validated immutable source snapshot. Observations and frames are source records; an observation epoch, spectral channel, broad wavelength band, and detector array are distinct dimensions.

## 4. Canonical status values



Reconcile with `SYSTEM_ARCHITECTURE.md`, `INVESTIGATION_WORKFLOW.md`, and `API_SPECIFICATION.md` before implementation.



**Task status:** `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRYING`, `CANCELLED`, `BLOCKED`, `NOT_SUPPORTED`, `INSUFFICIENT_DATA`.



**Quality outcome:** `PASSED`, `FAILED`, `INCONCLUSIVE`, `NOT_AVAILABLE`, `NOT_APPLICABLE`.



**Evidence gate:** `PASSED`, `FAILED`, `UNKNOWN`, `NOT_APPLICABLE`.



**Candidate status:** `NEEDS_MORE_DATA`, `NOT_PRIORITIZED`, `PRIORITIZED_FOR_REVIEW`, `BLOCKED_BY_QUALITY_ISSUE`.



Investigation, execution, retrieval, report, review, and agent-run statuses must use separately documented controlled values. For PostgreSQL, validate enums in application code and enforce stable values with `CHECK` constraints or PostgreSQL enum types where the migration policy justifies them. Prefer `CHECK` constraints when controlled values may evolve without an enum-type migration. Do not invent synonyms such as `SUCCESS` or `DONE` for canonical task states. A successful zero-result query is not a failed query. `SUCCEEDED` requires valid persisted output, not merely plausible agent prose.



## 5. Table specifications



Fields below describe the logical contract. Final SQLAlchemy types and nullability must be encoded and tested in migrations. Shared `created_at` fields are UTC timestamps. `JSON` means validated, bounded, non-secret structured data.



### 5.1 `investigations`

One user-requested scientific investigation.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `title` | Yes | Short user-facing title |
| `user_query` | Yes | Original request |
| `status` | Yes | Investigation lifecycle status; separate from task/execution status |
| `origin_type` | Yes | `DIRECT` or `EXPLORER`; records how the investigation was initiated |
| `requested_by` | No | Internal actor/user reference if authentication is in scope |
| `input_parameters_json` | Yes | Validated region, time range, dataset, and request parameters |
| `data_source_config_json` | Yes | Source identifiers and non-secret settings |
| `scope_summary` | No | Agreed scope summary |
| `created_at`, `updated_at` | Yes | UTC timestamps |
| `archived_at` | No | Optional soft-archive timestamp |

Retain the original query and validated input. Never put API keys in source configuration. For `origin_type = EXPLORER`, an investigation must reference at least one immutable server-validated source snapshot before an execution can start. Creating the investigation/snapshot does not itself create an execution or agent run. Do not cascade-delete scientific history.


### 5.2 `executions`

Each run of an investigation. A re-run creates a new row.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `investigation_id` | Yes | FK → `investigations.id` |
| `source_snapshot_id` | Conditional | FK → `investigation_source_snapshots.id`; required for an Explorer-origin execution |
| `run_number` | Yes | Monotonic run number per investigation |
| `status` | Yes | Canonical execution lifecycle status |
| `workflow_version` | Yes | Workflow definition version/hash |
| `config_snapshot_json` | Yes | Validated configuration snapshot |
| `started_at`, `finished_at` | No | UTC execution times |
| `failure_code`, `failure_message` | No | Stable code and safe diagnostic |
| `created_at` | Yes | UTC creation time |

Constraints: unique `(investigation_id, run_number)`. Index `(investigation_id, created_at)`. An Explorer-origin execution must reference the exact validated source snapshot used by that run. The referenced snapshot must belong to the same investigation; enforce this through a composite FK where practical or a transactionally enforced service invariant. Preserve prior runs and their evidence.


### 5.3 `tasks`



A unit of work managed by the workflow orchestrator.



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `execution_id` | Yes | FK → `executions.id` |

| `task_key` | Yes | Stable key within execution |

| `task_type`, `display_name` | Yes | Controlled task type and UI label |

| `status` | Yes | Canonical task status |

| `attempt_count`, `max_attempts` | Yes | Bounded retry tracking |

| `input_json` | Yes | Validated non-secret input |

| `output_summary_json` | No | Small structured summary; not large payloads |

| `failure_code`, `failure_message` | No | Stable failure and safe diagnostic |

| `started_at`, `finished_at` | No | UTC timestamps |

| `created_at`, `updated_at` | Yes | UTC timestamps |



Constraints: unique `(execution_id, task_key)`, `attempt_count >= 0`, `max_attempts >= 1`. Validate transitions in orchestration logic. Task outputs should reference persisted results/evidence.



### 5.4 `task_dependencies`



| Field | Required | Meaning |

|---|---:|---|

| `task_id` | Yes | FK → `tasks.id` |

| `depends_on_task_id` | Yes | FK → `tasks.id` |



Composite primary key `(task_id, depends_on_task_id)`. Disallow self-dependency; both tasks must belong to the same execution. Detect cycles before execution. Failed mandatory prerequisites block dependent tasks according to workflow policy.



### 5.5 `data_retrievals`

Records a fixture or external-source retrieval attempt. A retrieval may originate from a standalone Public Explorer query or an investigation execution.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `execution_id` | Conditional | FK → `executions.id`; null for a standalone Explorer retrieval |
| `explorer_query_id` | Conditional | FK → `explorer_queries.id`; set when retrieval belongs to a Public Explorer query |
| `task_id` | No | FK → `tasks.id` for investigation-owned retrieval work |
| `source_name` | Yes | Adapter/source identifier |
| `source_release`, `product_id` | No | Release and product identifier when actually known |
| `query_parameters_json` | Yes | Sanitized query parameters |
| `status` | Yes | Retrieval outcome, separate from task status |
| `result_count` | No | Actual count if known; non-negative |
| `empty_result` | No | True only for a confirmed successful zero-result query |
| `request_reference` | No | Safe source/query reference |
| `retrieved_at` | No | Actual retrieval timestamp |
| `content_checksum` | No | Checksum only if computed |
| `result_artifact_id` | No | FK → `artifacts.id` for a persisted retrieval artifact |
| `metadata_json` | No | Sanitized source metadata |
| `failure_code`, `failure_message` | No | Stable failure and safe diagnostic |
| `created_at` | Yes | UTC timestamp |

Require at least one owning context: `execution_id` or `explorer_query_id`. Both may be present only when a retrieval is explicitly reused/associated with an investigation under documented service rules. If `task_id` is set, it must belong to the referenced execution. Never fabricate archive IDs, timestamps, counts, or checksums. Clearly distinguish fixtures from live-source results. A successful zero-result query differs from a failed query.


### 5.6 `observations`

Normalized source observation/product record. An observation belongs to a retrieval, not intrinsically to an investigation execution, so Public Explorer can use it independently.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `data_retrieval_id` | Yes | FK → `data_retrievals.id`; retrieval connects the observation to an Explorer query or execution |
| `source_observation_id` | No | Original mission/archive identifier |
| `source_name`, `product_type` | Yes | Source and documented product kind |
| `data_release`, `product_id` | No | Exact release and product identifier when known |
| `observed_at` | No | Actual observation time, only if supplied and semantically valid |
| `ra_deg`, `dec_deg` | No | Coordinates where available |
| `coordinate_frame`, `coordinate_epoch` | No | Documented coordinate reference and epoch |
| `exposure_seconds` | No | Actual exposure duration if supplied |
| `quality_flags_json` | No | Original source flags |
| `source_artifact_id` | No | FK → `artifacts.id` for the original product when stored/controlled |
| `content_checksum` | No | Computed checksum for the source product when available |
| `metadata_json` | No | Validated source-specific metadata |
| `created_at` | Yes | UTC record creation time |

Do not infer observation time from retrieval time. An observation is not automatically one display frame. Do not treat SPHEREx spectral channels, broad wavelength bands, or detector arrays as separate observation epochs. Preserve the actual source observation ID, product ID, release, processing version, coordinate frame, and time metadata. Do not impose uniqueness on nullable source IDs without understanding source identity scope.


### 5.7 `measurements`

Immutable deterministic scientific result tied to explicit inputs and a versioned method.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `execution_id` | Yes | FK → `executions.id` |
| `task_id` | Yes | FK → `tasks.id` |
| `observation_id` | No | FK → `observations.id` when observation-specific |
| `frame_id` | No | FK → `observation_frames.id` when the measurement is frame/plane-specific |
| `measurement_key` | Yes | Stable measurement type/key |
| `value_numeric`, `value_text` | Conditional | Numeric or non-numeric value representation |
| `unit` | Conditional | Explicit unit for unit-bearing values |
| `uncertainty_numeric` | No | Uncertainty when available |
| `uncertainty_type` | No | Documented uncertainty meaning |
| `method_name`, `method_version` | Yes | Algorithm and exact version |
| `parameters_json` | Yes | Validated method parameters |
| `input_references_json` | Yes | Source input identifiers |
| `quality_status` | Yes | Quality outcome |
| `limitations_json` | No | Limitations or reason uncertainty is unavailable |
| `created_at` | Yes | UTC timestamp |

Require at least one value representation and validate combinations. Reject NaN/infinity as valid scientific outputs. Unavailable uncertainty is not zero. A frame-specific result must reference the correct observation/frame pair. Preserve old results when inputs, releases, parameters, method, or software change. LLMs do not calculate authoritative measurements.


### 5.8 `quality_checks`



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `execution_id` | Yes | FK → `executions.id` |

| `task_id`, `observation_id`, `frame_id`, `measurement_id` | No | Optional target FKs; at least one analysis target or a documented execution-level scope |

| `check_key`, `check_version` | Yes | Check identity and version |

| `outcome` | Yes | Canonical quality outcome |

| `summary` | Yes | Reason for outcome |

| `metrics_json`, `thresholds_json` | No | Actual diagnostics and thresholds |

| `created_at` | Yes | UTC timestamp |



`NOT_AVAILABLE` is not a pass. `NOT_APPLICABLE` requires a reason. Persist thresholds and check version.



### 5.9 `evidence_records`



Append-only evidence ledger. Evidence may represent source facts, measurements, quality results, catalogue results, statistical tests, alternative-hypothesis evaluations, or clearly labelled agent interpretations.



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `execution_id` | Yes | FK → `executions.id` |

| `task_id` | No | FK → `tasks.id` |

| `evidence_type`, `origin_type` | Yes | Controlled evidence type and origin (source, deterministic method, catalogue, agent interpretation) |

| `title`, `summary` | Yes | Human-readable, bounded description |

| `payload_json` | No | Validated structured payload |

| `source_reference` | No | Source/product/document reference |

| `method_name`, `method_version` | No | Derivation method and version |

| `input_hash` | No | Hash of canonicalized inputs when computed |

| `supersedes_evidence_id` | No | FK → `evidence_records.id` for explicit supersession |

| `created_at` | Yes | UTC timestamp |



Separate observations, measurements, and agent interpretations. Quantitative claims must link to evidence. Never represent agent text as a direct observation. Missing provenance remains explicit.



### 5.10 `evidence_links`

Typed relationships between evidence and the entities/results that support or qualify it.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `evidence_id` | Yes | FK → `evidence_records.id` |
| `related_evidence_id` | No | FK → `evidence_records.id` |
| `related_measurement_id` | No | FK → `measurements.id` |
| `related_observation_id` | No | FK → `observations.id` |
| `related_frame_id` | No | FK → `observation_frames.id` |
| `related_artifact_id` | No | FK → `artifacts.id` |
| `related_frame_manifest_id` | No | FK → `frame_manifests.id` |
| `related_source_snapshot_id` | No | FK → `investigation_source_snapshots.id` |
| `related_quality_check_id` | No | FK → `quality_checks.id` |
| `relation_type` | Yes | Controlled relation, e.g. `DERIVED_FROM`, `SUPPORTS`, `CONTRADICTS`, `CONTEXTUALIZES` |
| `created_at` | Yes | UTC timestamp |

Each link must identify at least one related target. Enforce this with a PostgreSQL `CHECK` requiring at least one typed target FK. Avoid unconstrained polymorphic IDs. Add explicit typed FKs or a dedicated typed-link table when more targets are needed.


### 5.11 `candidates`



A source/object under review, never a confirmed discovery.



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `investigation_id` | Yes | FK → `investigations.id` |

| `execution_id` | Yes | FK → `executions.id` |

| `candidate_key` | Yes | Stable key within execution |

| `label` | No | Human-readable label |

| `ra_deg`, `dec_deg`, `coordinate_frame` | No | Supported coordinates and frame |

| `status` | Yes | Canonical candidate status |

| `priority_score` | No | Deterministic score only if policy-produced |

| `scoring_policy_version` | No | Exact scoring policy version |

| `rationale_summary` | No | Evidence-linked rationale summary |

| `created_at` | Yes | UTC timestamp |



Unique `(execution_id, candidate_key)`. A catalogue no-match is not proof of novelty. Do not use statuses such as “planet confirmed.”



### 5.12 `candidate_assessments`



Versioned assessment of a candidate against a policy and evidence set.



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `candidate_id` | Yes | FK → `candidates.id` |

| `execution_id` | Yes | FK → `executions.id` |

| `assessment_version` | Yes | Version per candidate |

| `status` | Yes | Canonical candidate status |

| `score` | No | Deterministic score if applicable |

| `policy_name`, `policy_version` | Yes | Assessment policy and version |

| `criteria_results_json` | Yes | Criterion outcomes and evidence references |

| `gate_results_json` | Yes | Gate outcomes and reasons |

| `alternative_hypotheses_json` | No | Structured alternatives and evidence links |

| `missing_evidence_json` | No | Explicit gaps |

| `created_at` | Yes | UTC timestamp |



Unique `(candidate_id, assessment_version)`. A score is not automatically a calibrated probability. Human review is separate from automated prioritization.



### 5.13 `catalogue_matches`

Persist catalogue query outcomes and associations when catalogue matching is part of the selected workflow.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `execution_id` | Yes | FK → `executions.id` |
| `task_id`, `candidate_id` | No | Optional FKs |
| `target_resolution_id` | No | FK → `target_resolutions.id` when a named target was resolved before matching |
| `catalogue_name` | Yes | Catalogue identifier |
| `catalogue_release` | No | Release when known |
| `query_parameters_json` | Yes | Actual coordinates/radius/filters, coordinate frame/epoch, and query criteria |
| `query_status` | Yes | Controlled query outcome distinguishing success-with-match, successful-empty, failure, and unsupported operation |
| `matched_object_id` | No | Returned catalogue ID |
| `separation_arcsec` | No | Computed separation if available |
| `match_method` | No | Method/version |
| `match_quality_json` | No | Ambiguity and match criteria |
| `source_reference` | No | Query/result reference |
| `created_at` | Yes | UTC timestamp |

Empty result and failed query must remain distinct. Store radius, coordinate frame/epoch, catalogue release, and algorithm when known. Gaia and WISE/NEOWISE remain separately attributed catalogue sources; do not merge their rows into SPHEREx observations. A no-match does not establish novelty.


### 5.14 `agent_runs`



Metadata for an LLM/agent invocation. Agent output is not scientific measurement.



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `execution_id` | Yes | FK → `executions.id` |

| `task_id` | No | FK → `tasks.id` |

| `agent_role` | Yes | One of the documented logical agent roles |

| `provider_name`, `model_name` | No | Provider/model identifiers if known |

| `prompt_version` | Yes | Prompt/template version or hash |

| `input_schema_version`, `output_schema_version` | Yes | Structured contract versions |

| `status` | Yes | Agent-run outcome |

| `input_evidence_ids_json` | Yes | Evidence IDs supplied to the agent |

| `output_json` | No | Validated structured output |

| `validation_errors_json` | No | Schema/grounding errors |

| `attempt_count` | Yes | Bounded attempts |

| `started_at`, `finished_at` | No | UTC timestamps |

| `created_at` | Yes | UTC timestamp |



Do not persist hidden chain-of-thought. Store concise evidence-linked rationale, structured outputs, and validation outcomes only. Agent runs cannot mutate measurements or bypass gates. Avoid secrets and unnecessary personal data in inputs/outputs.



### 5.15 `reports`



Versioned report for an execution.



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `investigation_id` | Yes | FK → `investigations.id` |

| `execution_id` | Yes | FK → `executions.id` |

| `report_version` | Yes | Version per execution |

| `status` | Yes | Report lifecycle status |

| `title` | Yes | Report title |

| `summary` | No | Evidence-bounded summary |

| `content_json` | Yes | Structured report sections and references |

| `format` | Yes | Supported output format |

| `template_version` | Yes | Report template version |

| `generated_at` | Yes | UTC timestamp |

| `artifact_id` | No | FK → `artifacts.id` for a controlled generated report/export artifact |
| `content_checksum` | No | Checksum only when computed |

| `created_at` | Yes | UTC timestamp |



Unique `(execution_id, report_version)`. Regenerating a report creates a new version; it does not silently replace a previously reviewed report. Reports must use persisted evidence, not agent memory alone.



### 5.16 `report_claims`



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `report_id` | Yes | FK → `reports.id` |

| `claim_key` | Yes | Stable key within report |

| `claim_text` | Yes | Bounded claim |

| `claim_type` | Yes | Observation, measurement, interpretation, limitation, or recommendation |

| `evidence_ids_json` | Yes | Valid evidence IDs, checked against evidence store |

| `qualification` | No | Explicit uncertainty/qualification; no uncalibrated probability claims |

| `created_at` | Yes | UTC timestamp |



Quantitative claims must cite supporting measurements/evidence. If strict FK integrity is required, replace `evidence_ids_json` with a join table `report_claim_evidence`.



### 5.17 `human_reviews`



| Field | Required | Meaning |

|---|---:|---|

| `id` | Yes | UUID primary key |

| `investigation_id` | Yes | FK → `investigations.id` |

| `execution_id`, `candidate_id`, `report_id` | At least one | Optional FKs for the reviewed target(s) |

| `reviewer_id` | No | Internal user reference if authentication exists |

| `decision` | Yes | Controlled review decision |

| `rationale` | Yes | Human rationale |

| `reviewed_evidence_version` | No | Execution/report/evidence version reviewed |

| `reviewed_at`, `created_at` | Yes | UTC timestamps |



Enforce that at least one target FK is present. Reviews are append-only: a new decision creates a new row. Accepting a report does not confirm its scientific hypothesis.



### 5.18 Optional later entities



**`alternative_hypothesis_evaluations`:** add only when the first supported workflow defines a structured contract. Suggested fields: `id`, `execution_id`, optional `candidate_id`, `hypothesis_key`, `hypothesis_version`, `status`, `evidence_for_json`, `evidence_against_json`, `tests_performed_json`, `limitations_json`, `created_at`. Otherwise store these as typed evidence records for the MVP.



**`users` / `workspaces`:** do not add unless authentication and tenancy are in the agreed MVP scope. If added, define ownership and authorization before exposing data. UUIDs are not access control.




### 5.19 `artifacts`

Central registry for controlled files stored outside PostgreSQL. An artifact row describes metadata and access state; it does not contain the FITS/image/spectrum/report binary itself.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `artifact_type` | Yes | Controlled type such as `SOURCE_PRODUCT`, `CUTOUT`, `PREVIEW`, `MASK`, `VARIANCE`, `COMPARISON`, `SPECTRUM`, or `REPORT` |
| `storage_backend` | Yes | Configured storage adapter name, not a client-controlled provider |
| `storage_key` | Yes | Private opaque object key/reference; never exposed as a public filesystem path |
| `media_type` | Yes | Validated MIME/media type |
| `byte_size` | No | Actual byte size when known; non-negative |
| `checksum_algorithm`, `checksum_value` | No | Algorithm and checksum only when actually computed |
| `width_px`, `height_px` | No | Image dimensions when applicable |
| `pixel_scale_value`, `pixel_scale_unit` | No | Pixel scale when known |
| `storage_status` | Yes | Controlled artifact lifecycle state, e.g. `PENDING`, `AVAILABLE`, `FAILED`, `EXPIRED`, `DELETED` |
| `source_reference` | No | Safe source-product reference, not a credential or unrestricted proxy URL |
| `derivation_metadata_json` | No | Validated input IDs, cutout parameters, processing version, and derivation provenance |
| `expires_at` | No | Expiry when temporary |
| `created_at`, `updated_at` | Yes | UTC timestamps |

Constraints: unique `(storage_backend, storage_key)`; `byte_size >= 0` when present; non-negative dimensions when present. `AVAILABLE` requires a resolvable object according to the storage adapter. Artifact-content APIs must authorize and stream/redirect by artifact ID; they must never expose `storage_key`, credentials, or arbitrary client-provided URLs. Deleting an artifact row must not silently delete evidence history; storage deletion follows an explicit retention workflow.

### 5.20 `explorer_queries`

Bounded, read-only Public Explorer sky-region query record. It is not an investigation and must not create an execution, agent run, candidate, or human review.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `query_version` | Yes | Query request/normalization contract version |
| `status` | Yes | Canonical operation/task status used by the API |
| `ra_deg`, `dec_deg` | Yes | Validated normalized sky centre |
| `coordinate_frame` | Yes | Explicit coordinate frame/convention |
| `fov_width_value`, `fov_height_value` | Yes | Validated FOV dimensions |
| `fov_unit` | Yes | Unit for FOV values |
| `time_start`, `time_end` | No | Requested time bounds; not inferred from the query time |
| `source_filters_json` | No | Bounded source/product/release filters |
| `normalized_query_json` | Yes | Canonical non-secret query parameters used for reproducibility/cache key |
| `observation_count` | No | Actual number of returned observations/products when known; non-negative |
| `frame_count` | No | Actual number of normalized display frames when known; non-negative |
| `empty_result` | No | True only when the completed query returned zero qualifying observations; this does not mean no object exists |
| `failure_code`, `failure_message` | No | Stable safe error details |
| `anonymous_session_hash` | No | Optional non-reversible, short-lived abuse-control reference; never store raw session secrets |
| `requested_at`, `completed_at` | No | UTC query lifecycle timestamps |
| `expires_at` | No | Optional metadata retention expiry |
| `created_at` | Yes | UTC record creation time |

A successful empty query is not a failed query. `observation_count = 0` and `empty_result = true` are valid only after successful completion. A query may return observations but zero displayable frames; preserve those counts separately. Do not persist every public request indefinitely; apply documented retention, rate limits, and privacy controls. Query records must not be used to infer that an object does not exist.

### 5.21 `frame_manifests`

Versioned manifest for displayable frames returned by an Explorer query. Manifest membership is recorded relationally in `frame_manifest_items`; a JSON copy may be retained for reproducibility but is not the only relationship record.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `explorer_query_id` | Yes | FK → `explorer_queries.id` |
| `manifest_version` | Yes | Monotonic version for the query |
| `schema_version` | Yes | Manifest schema version |
| `status` | Yes | Canonical operation/task status or a documented manifest lifecycle value |
| `frame_count` | Yes | Actual number of manifest members; non-negative |
| `sequence_count` | No | Actual number of compatible display sequences when computed |
| `manifest_hash` | No | Hash only when computed from canonical manifest content |
| `ordering_rule` | Yes | Must identify ordering by actual observation time within a compatible sequence |
| `selection_metadata_json` | No | Applied compatibility/grouping filters and display metadata |
| `created_at` | Yes | UTC timestamp |

Constraints: unique `(explorer_query_id, manifest_version)`. Do not manufacture timestamps or include a frame in a sequence when compatibility rules fail. The manifest is a display/data-selection object; it is not itself scientific proof of motion or variability.

### 5.22 `observation_frames`

A normalized image plane or frame-level product associated with a source observation/product. This entity separates the actual observation epoch from spectral-channel, broad-band, and detector-array dimensions.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `observation_id` | Yes | FK → `observations.id` |
| `source_frame_id` | No | Original archive/product frame/plane identifier when supplied |
| `frame_key` | Yes | Stable key within the normalized observation |
| `frame_type` | Yes | Controlled product/frame kind, e.g. `SOURCE_IMAGE`, `CHANNEL_PLANE`, `ARRAY_PRODUCT`, or `DISPLAY_DERIVATIVE` |
| `observed_at` | No | Actual frame observation time only when the source product defines it; never retrieval time |
| `channel_index` | No | Actual channel index when supplied by the product |
| `channel_id` | No | Source/product channel identifier |
| `wavelength_center` | No | Channel wavelength centre when defined |
| `wavelength_min`, `wavelength_max` | No | Channel wavelength bounds when defined |
| `wavelength_unit` | No | Explicit wavelength unit |
| `broad_band_id` | No | Broad-band identifier, stored separately from channel identity |
| `detector_array_id` | No | Detector-array identifier, stored separately from channel and band |
| `product_type`, `product_release` | No | Exact product type/release where supplied |
| `coordinate_frame`, `wcs_metadata_json` | No | WCS/coordinate metadata and validated header subset |
| `width_px`, `height_px` | No | Actual pixel dimensions when available |
| `pixel_scale_value`, `pixel_scale_unit` | No | Pixel scale when available |
| `data_unit` | No | Unit represented by pixel values, if known |
| `calibration_state` | No | Documented calibration state; do not infer calibrated status |
| `quality_flags_json` | No | Original product flags and normalized interpretation references |
| `source_artifact_id` | No | FK → `artifacts.id` for original/source product or plane |
| `cutout_artifact_id` | No | FK → `artifacts.id` for validated cutout |
| `preview_artifact_id` | No | FK → `artifacts.id` for display-only preview |
| `mask_artifact_id` | No | FK → `artifacts.id` when supplied |
| `variance_artifact_id` | No | FK → `artifacts.id` when supplied |
| `metadata_json` | No | Bounded validated product-specific metadata |
| `created_at` | Yes | UTC record creation time |

Constraints: unique `(observation_id, frame_key)`; positive dimensions when present; wavelength bounds must be ordered; channel index must be non-negative when present. A channel is not an epoch. A broad band is not a channel index. A detector array is not a time sample. Do not hard-code the presence or count of 102 channels, six broad bands, or six arrays for every release/product; persist what the selected product actually provides. Preserve original frames and epochs; any validated coadd is a separate derived artifact/operation and must not replace them.

### 5.23 `frame_manifest_items`

Membership and ordering of frames within a specific immutable manifest version.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `frame_manifest_id` | Yes | FK → `frame_manifests.id` |
| `frame_id` | Yes | FK → `observation_frames.id` |
| `sequence_key` | Yes | Compatible sequence grouping key within the manifest |
| `sequence_order` | Yes | Display order within the sequence |
| `observed_at_snapshot` | No | Actual observation timestamp copied at manifest creation for stable ordering/audit |
| `compatibility_metadata_json` | No | Compatibility/grouping rationale and selected channel/band/array dimensions |
| `created_at` | Yes | UTC timestamp |

Constraints: unique `(frame_manifest_id, frame_id)` and unique `(frame_manifest_id, sequence_key, sequence_order)`. The frame's source record remains authoritative; the timestamp snapshot preserves the manifest as created. The service must verify that each frame belongs to the query's retrieved source set.

### 5.24 `investigation_source_snapshots`

Immutable server-validated record of the Explorer selections used to create an Explorer-origin investigation. A new or changed selection creates a new snapshot version; it must not rewrite a prior snapshot.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `investigation_id` | Yes | FK → `investigations.id` |
| `snapshot_version` | Yes | Monotonic version per investigation |
| `explorer_query_id` | Yes | FK → `explorer_queries.id` |
| `frame_manifest_id` | Yes | FK → `frame_manifests.id` |
| `manifest_version` | Yes | Exact manifest version selected by the user |
| `normalized_region_json` | Yes | Server-normalized sky centre, coordinate frame, and FOV |
| `query_parameters_snapshot_json` | Yes | Query filters/time constraints used to produce the selection |
| `source_provenance_snapshot_json` | Yes | Source/product/release/processing identifiers and relevant quality metadata |
| `snapshot_hash` | Yes | Hash of canonicalized selected references and metadata, computed by the application |
| `created_at` | Yes | UTC timestamp |

Constraints: unique `(investigation_id, snapshot_version)` and unique `snapshot_hash` within the applicable investigation scope where appropriate. Snapshot rows and items are append-only. The server, not the browser, resolves observation/product details and computes the hash. Client-supplied descriptions, pixel values, URLs, or coordinates cannot override source records.

### 5.25 `investigation_source_snapshot_items`

Explicit selected observations/frames captured by a source snapshot.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `source_snapshot_id` | Yes | FK → `investigation_source_snapshots.id` |
| `observation_id` | Yes | FK → `observations.id` |
| `frame_id` | No | FK → `observation_frames.id`; null when the selection intentionally refers to the observation/product as a whole |
| `selection_order` | Yes | Stable user/server selection order |
| `selected_metadata_snapshot_json` | Yes | Release, product, observation time, channel/band/array identifiers, and relevant metadata as validated at snapshot creation |
| `created_at` | Yes | UTC timestamp |

Constraints: unique `(source_snapshot_id, selection_order)`; add a partial unique index on `(source_snapshot_id, frame_id)` where `frame_id IS NOT NULL` and a partial unique index on `(source_snapshot_id, observation_id)` where `frame_id IS NULL` to prevent duplicate selections. Validate at service level that `frame_id`, when present, belongs to `observation_id`, and that all selected frames are members of the referenced manifest. At least one item is required before an Explorer-origin snapshot can be used to start an execution.

### 5.26 `source_snapshot_validations`

Append-only record of revalidation performed before an execution consumes an Explorer snapshot. The snapshot itself remains immutable.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `source_snapshot_id` | Yes | FK → `investigation_source_snapshots.id` |
| `execution_id` | No | FK → `executions.id`; set when validation is associated with a run request |
| `outcome` | Yes | Controlled result: `PASSED`, `FAILED`, or `INCONCLUSIVE` |
| `validated_manifest_version` | No | Actual version checked |
| `validated_release_metadata_json` | No | Current source/release/reference validation results |
| `discrepancies_json` | No | Missing, stale, mismatched, or unavailable items |
| `summary` | Yes | Human-readable validation outcome |
| `validated_at` | Yes | UTC timestamp |

A failed or inconclusive mandatory source-snapshot validation must block the dependent run. Do not silently replace the selected observations/frames with newer results. A user may return to Explorer and create a new snapshot.

### 5.27 `task_attempts`

Append-only record of each concrete task attempt so retry history is preserved, rather than relying only on a mutable counter on `tasks`.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `task_id` | Yes | FK → `tasks.id` |
| `attempt_number` | Yes | One-based attempt number |
| `status` | Yes | Canonical task status for this attempt |
| `input_hash` | No | Hash of canonicalized inputs when computed |
| `output_summary_json` | No | Small validated summary/result reference |
| `failure_code`, `failure_message` | No | Stable code and safe diagnostic |
| `started_at`, `finished_at` | No | UTC timestamps |
| `created_at` | Yes | UTC timestamp |

Constraints: unique `(task_id, attempt_number)`; `attempt_number >= 1`. `tasks.attempt_count` may remain as a denormalized aggregate, but it must agree with persisted attempts. A retry creates another attempt; it does not erase the earlier attempt or its evidence.

### 5.28 `idempotency_records`

Durable idempotency registry for API operations such as investigation creation, run creation, Explorer query submission, report generation, and other supported mutations.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `scope_key` | Yes | Authenticated actor/workspace scope or a documented anonymous operation scope |
| `idempotency_key` | Yes | Client-supplied key, validated and size-bounded |
| `request_hash` | Yes | Hash of canonicalized effective request; do not store secrets or unrestricted raw payloads |
| `resource_type` | No | Created/affected resource type |
| `resource_id` | No | Created/affected resource ID |
| `http_status` | No | Original operation response status |
| `response_metadata_json` | No | Minimal replayable response metadata, not large scientific content or credentials |
| `created_at` | Yes | UTC creation time |
| `expires_at` | Yes | UTC retention expiry |

Constraints: unique `(scope_key, idempotency_key)`; expiry must be after creation. Same key + same request hash returns the original operation/resource result; same key + different hash returns `409 IDEMPOTENCY_CONFLICT`. Define retention and cleanup policy. Idempotency records are operational metadata and must not replace domain records.

### 5.29 `image_comparisons` (conditional P1)

Persist only when a validated server-side image comparison operation is enabled. Frontend blinking/playback does not require this table.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `execution_id` | No | FK → `executions.id`; null for a standalone Explorer comparison job |
| `explorer_query_id` | No | FK → `explorer_queries.id` when initiated from Explorer |
| `operation_type` | Yes | Controlled operation such as `DIFFERENCE`, `ADDITION`, `STACK`, or validated `ARRAY_COMBINATION` |
| `status` | Yes | Canonical operation/task status |
| `method_name`, `method_version` | Yes | Exact processing method/software version |
| `parameters_json` | Yes | Validated operation parameters |
| `compatibility_outcome` | Yes | Quality/compatibility outcome |
| `diagnostics_json` | No | Registration, PSF, background, mask, variance, calibration, and residual diagnostics when available |
| `output_artifact_id` | No | FK → `artifacts.id` |
| `limitations_json` | No | Explicit caveats and unavailable prerequisites |
| `created_at`, `completed_at` | No | UTC operation times |

At least one context (`execution_id` or `explorer_query_id`) is required. A completed status requires a valid output artifact or a documented no-file result schema. Do not mix incompatible releases, units, channels, arrays, calibration states, or epochs. Cross-epoch stacking must not replace original epoch frames and must be blocked when it would invalidate the scientific objective.

### 5.30 `image_comparison_inputs` (conditional P1)

| Field | Required | Meaning |
|---|---:|---|
| `image_comparison_id` | Yes | FK → `image_comparisons.id` |
| `frame_id` | Yes | FK → `observation_frames.id` |
| `input_role` | Yes | Controlled role such as `REFERENCE`, `SCIENCE`, or `STACK_MEMBER` |
| `input_order` | Yes | Stable order |
| `created_at` | Yes | UTC timestamp |

Composite primary key `(image_comparison_id, frame_id)` and unique `(image_comparison_id, input_order)`. The exact input frames and order must be retained for reproducibility.

### 5.31 `target_resolutions` (optional P1)

Authoritative name-to-coordinate resolution record for named challenge targets or user-supplied object names. Do not invent coordinates.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `target_name_query` | Yes | Original user-supplied name/query |
| `resolver_name` | Yes | Resolver/catalogue identifier |
| `resolver_release` | No | Resolver/catalogue release when known |
| `query_parameters_json` | Yes | Actual resolver query and disambiguation parameters |
| `resolution_status` | Yes | Controlled value such as `RESOLVED`, `AMBIGUOUS`, `NO_RESULT`, `FAILED` |
| `resolved_identifier` | No | Exact returned object identifier/component |
| `ra_deg`, `dec_deg` | No | Returned coordinates only when resolved |
| `coordinate_frame`, `coordinate_epoch` | No | Coordinate reference metadata |
| `result_metadata_json` | No | Returned identifiers, alternatives, and source details |
| `source_reference` | No | Traceable resolver result/document reference |
| `created_at` | Yes | UTC timestamp |

Ambiguous components (for example, multiple components of a named system) must remain ambiguous until explicitly resolved. A no-result resolver response is not proof that the object does not exist.

### 5.32 `derived_spectra` (conditional P1)

Metadata record for an on-demand derived spectrum. Store large spectral arrays as controlled artifacts, not as an unbounded JSON row.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `execution_id` | No | FK → `executions.id`; null for standalone Explorer operation where supported |
| `explorer_query_id` | No | FK → `explorer_queries.id` for a standalone Explorer-origin extraction |
| `observation_id` | Yes | FK → `observations.id` |
| `method_name`, `method_version` | Yes | Validated extraction method and version |
| `wavelength_unit`, `flux_unit` | Yes | Explicit output units |
| `artifact_id` | Yes | FK → `artifacts.id` for the derived data product |
| `uncertainty_artifact_id` | No | FK → `artifacts.id` when uncertainty is supplied separately |
| `input_frames_json` | Yes | Frame IDs and exact selection, validated against the frame registry |
| `parameters_json` | Yes | Extraction parameters and aperture/background/calibration choices as applicable |
| `quality_outcome` | Yes | Canonical quality outcome |
| `limitations_json` | No | Missing calibration, channels, or method limitations |
| `created_at` | Yes | UTC timestamp |

Require at least one context (`execution_id` or `explorer_query_id`); if both are set, enforce that the execution uses the same source context. Do not assume a spectrum can be extracted from every observation or product. Do not interpret wavelength samples as epochs. A fitted physical property requires its own validated model and uncertainty provenance.

### 5.33 `external_overlay_queries` (optional P1)

Provenance record for separate contextual overlays such as Gaia or WISE/NEOWISE. These catalogues remain distinct from SPHEREx source observations.

| Field | Required | Meaning |
|---|---:|---|
| `id` | Yes | UUID primary key |
| `explorer_query_id` | Yes | FK → `explorer_queries.id` |
| `catalogue_name` | Yes | External catalogue identifier |
| `catalogue_release` | No | Exact release/version when known |
| `query_parameters_json` | Yes | Actual region, epoch, filters, and query parameters |
| `coordinate_frame`, `coordinate_epoch` | No | Coordinate metadata |
| `result_artifact_id` | No | FK → `artifacts.id` for VOTable/table/overlay payload when persisted |
| `result_count` | No | Actual returned count when known |
| `status` | Yes | Documented query operation status |
| `limitations_json` | No | Cross-match, epoch, coverage, or catalogue-completeness limitations |
| `queried_at` | Yes | UTC query time |

Overlay rows must preserve their catalogue source, release, epoch, units, and coordinate reference. They must not be merged into SPHEREx observations or described as SPHEREx-native data.

### 5.34 `report_claim_evidence`

Relational join table between report claims and their supporting evidence. This replaces reliance on an unchecked JSON array when strict referential integrity is required.

| Field | Required | Meaning |
|---|---:|---|
| `report_claim_id` | Yes | FK → `report_claims.id` |
| `evidence_id` | Yes | FK → `evidence_records.id` |
| `relation_type` | Yes | Controlled relationship such as `SUPPORTS`, `CONTRADICTS`, or `QUALIFIES` |
| `created_at` | Yes | UTC timestamp |

Composite primary key `(report_claim_id, evidence_id, relation_type)`. Keep `report_claims.evidence_ids_json` only as a serialized API/report snapshot if required; the relational join is authoritative for database integrity.

## 6. Relationship and delete policy



| Parent | Child | Cardinality | Delete policy |

|---|---|---|---|

| explorer_queries | data_retrievals | 1-to-many | Restrict/retention policy |
| explorer_queries | frame_manifests | 1-to-many versions | Restrict/retention policy |
| frame_manifests | frame_manifest_items | 1-to-many | Restrict |
| observations | observation_frames | 1-to-many | Restrict |
| observations | measurements | 1-to-many where observation-specific | Restrict |
| observation_frames | measurements | 1-to-many where frame-specific | Restrict |
| frame_manifest_items | observation_frames | many-to-one | Restrict |
| investigations | investigation_source_snapshots | 1-to-many versions | Restrict |
| investigation_source_snapshots | executions | 1-to-many run references | Restrict |
| investigation_source_snapshots | snapshot_items | 1-to-many | Restrict |
| investigation_source_snapshots | source_snapshot_validations | 1-to-many | Restrict |
| investigations | executions | 1-to-many | Restrict/soft archive |

| executions | tasks | 1-to-many | Restrict |
| tasks | task_attempts | 1-to-many | Restrict |
| executions | source_snapshot_validations | 1-to-many | Restrict |

| tasks | task_dependencies | self many-to-many | Restrict; same-execution validation |

| executions | data_retrievals | 1-to-many where retrieval is investigation-owned | Restrict |

| data_retrievals | observations | 1-to-many | Restrict |


| executions | quality_checks | 1-to-many | Restrict |

| executions | evidence_records | 1-to-many | Restrict |

| evidence_records | evidence_links | 1-to-many | Restrict |

| investigations/executions | candidates | 1-to-many | Restrict |

| candidates | candidate_assessments | 1-to-many | Restrict |

| executions | agent_runs | 1-to-many | Restrict |

| executions | reports | 1-to-many | Restrict |
| artifacts | retrievals/observations/frames/comparisons/spectra/reports | referenced by typed FKs | Restrict |
| image_comparisons | image_comparison_inputs | 1-to-many | Restrict |
| report_claims | report_claim_evidence | 1-to-many | Restrict |

| reports | report_claims | 1-to-many | Restrict |

| investigations | human_reviews | 1-to-many | Restrict |



Avoid cascading deletion of scientific records. Any purge/retention workflow must be explicit, authorized, documented, and tested.



## 7. Constraints and indexes



### Required constraints



- Primary keys and foreign keys on all entities/relationships; use PostgreSQL native UUID columns.

- Unique `(investigation_id, run_number)` on executions.

- Unique `(execution_id, task_key)` on tasks.

- Unique `(execution_id, candidate_key)` on candidates.

- Unique `(candidate_id, assessment_version)` on candidate assessments.

- Unique `(execution_id, report_version)` on reports.

- `attempt_count >= 0`, `max_attempts >= 1`; validate non-negative uncertainty where appropriate.

- Validate coordinates when present: RA within the documented convention (commonly 0–360 degrees), declination -90 to +90 degrees.

- Reject invalid enum/status values and invalid lifecycle transitions.

- Validate that task dependencies connect tasks in the same execution and contain no cycles.
- Require each data retrieval to have an Explorer-query or execution context.
- Require Explorer-origin executions to reference a validated immutable source snapshot.
- Verify that manifest items belong to the manifest query and that snapshot-selected frames belong to the selected manifest/observation.
- Verify that an observation frame’s channel, broad-band, detector-array, and wavelength fields match the actual source-product metadata; do not conflate dimensions.
- Require artifact references to resolve through the controlled artifact registry.
- Require task-attempt numbers to be unique per task and preserve previous attempts.
- Enforce idempotency uniqueness by `(scope_key, idempotency_key)` and reject request-hash mismatches.
- Require each report claim’s evidence references to exist through `report_claim_evidence` when strict integrity is enabled.

- Validate evidence links have a target and human reviews target at least one entity.

- Validate report claim evidence IDs exist.



### Recommended indexes



- `explorer_queries(created_at, status)`
- Partial index on `explorer_queries(expires_at)` for retention cleanup where `expires_at` is not null
- `data_retrievals(explorer_query_id, source_name)` where `explorer_query_id` is not null
- `frame_manifests(explorer_query_id, manifest_version)`
- `observation_frames(observation_id, observed_at)`
- `observation_frames(channel_id, detector_array_id, broad_band_id)` where query patterns justify it
- `frame_manifest_items(frame_manifest_id, sequence_key, sequence_order)`
- `investigation_source_snapshots(investigation_id, snapshot_version)`
- `investigation_source_snapshot_items(source_snapshot_id, selection_order)`
- Partial unique indexes on selected non-null `frame_id` and observation-level null-frame snapshot items
- `executions(investigation_id, created_at)`

- `tasks(execution_id, status)`
- `task_attempts(task_id, attempt_number)`
- `idempotency_records(scope_key, idempotency_key)` unique

- `data_retrievals(execution_id, source_name)`

- `observations(data_retrieval_id, observed_at)`

- `observations(source_name, source_observation_id)` when identity semantics allow

- `measurements(execution_id, measurement_key)`

- `quality_checks(execution_id, outcome)`

- `evidence_records(execution_id, evidence_type, created_at)`

- `candidates(investigation_id, status)`

- `candidate_assessments(candidate_id, assessment_version)`

- `catalogue_matches(execution_id, catalogue_name)`

- `agent_runs(execution_id, agent_role, status)`

- `reports(execution_id, report_version)`
- `artifacts(storage_backend, storage_key)` unique
- `image_comparisons(execution_id, status)` where applicable
- `external_overlay_queries(explorer_query_id, catalogue_name, queried_at)` where applicable

- `human_reviews(investigation_id, reviewed_at)`



Choose indexes for real query patterns using PostgreSQL `EXPLAIN (ANALYZE, BUFFERS)` on representative data. Use partial indexes for nullable context FKs when useful. JSONB GIN indexes are opt-in and require a demonstrated filter/search path. Frequently filtered values should be first-class columns. Do not add spatial extensions or broad JSONB indexes pre-emptively.



### Idempotency



- Use stable `task_key` plus unique constraint to avoid duplicate task creation.

- Retried work must not duplicate committed outputs unintentionally.

- Preserve source product IDs and their identity scope when the source provides them.

- Distinguish retries from new analyses using execution, task, method version, and canonicalized inputs.

- A changed release, method, or parameter set must produce distinguishable results.



## 8. Provenance and reproducibility



Every quantitative report value should trace to:



1\. Source observation/product and archive identifier, when supplied.

2\. Data source and exact release/product version, when available.

3\. Retrieval parameters and outcome.

4\. Quality flags and check results.

5\. Deterministic method name/version and software version where relevant.

6\. Method parameters, units, coordinate conventions, and uncertainty treatment.

7\. Input measurement/evidence IDs.

8\. Execution and result version.

9\. Report claim that references the result.



Use explicit relational IDs for core entities. JSON snapshots preserve parameters and metadata but must not be the only traceability mechanism.



### Artifact references



- Store large products outside PostgreSQL; persist an `artifact_id`, private storage reference, source ID, media/product type, byte size if known, and checksum only if calculated.

- Do not expose private absolute paths, object-store keys, bucket names, or credentials in public API responses.

- Clearly label synthetic fixtures; never present them as live archive data.



### Versioning



Persist versions of workflow definitions, scientific methods/parameters, data releases, agent prompts and schemas (when used), candidate scoring policies, and report templates/content. Do not retain only a mutable “latest result.”



## 9. Security and privacy



- Never persist API keys, passwords, access tokens, or secrets in any table.

- Keep credentials in environment/secret configuration, not query JSON, metadata, reports, or error messages.

- Bound and validate JSON payloads.

- Do not store hidden chain-of-thought; store concise, evidence-linked rationale and validated structured output.

- Store only user identifiers needed by the agreed authorization model.

- Enforce authorization in services/API queries; UUIDs are not authorization.

- Do not expose stack traces, internal paths, or raw exceptions in API responses.

- Use separate PostgreSQL migration-owner and application-runtime roles; the runtime role must not have unrestricted DDL privileges.
- Enforce TLS for remote database connections and keep credentials in secret configuration.
- Document PostgreSQL backup/PITR, restore access, retention, and deletion rules.
- If multiple users/workspaces are enabled, enforce authorization in service queries; consider PostgreSQL row-level security only after the tenant/ownership model is explicitly defined and tested.

- If multi-user access is introduced, enforce ownership/tenant boundaries on every read and write.



## 10. Migrations

1. Reconcile this specification with the authoritative source documents before creating the initial migration. PostgreSQL is the selected final database; do not create a parallel SQLite migration path.
2. Keep Alembic migrations small, deterministic, reviewed, and PostgreSQL-tested.
3. Never edit an already-applied migration on a shared/persistent database; create a new revision.
4. Test migration from an empty PostgreSQL database and from every supported previous revision used by deployment.
5. Test PostgreSQL foreign keys, unique/check constraints, partial indexes, JSONB behavior, transaction rollback, and concurrency-sensitive constraints.
6. Back up persistent data and referenced artifacts before deployment migrations; verify the restore path.
7. Document how to inspect the current Alembic revision and recover from migration failure.
8. Never claim migration success unless commands ran against PostgreSQL and the database was verified.
9. Do not use SQLite-only test fixtures as the acceptance proof for PostgreSQL behavior. Unit tests may use isolated abstractions, but database integration and migration tests must target PostgreSQL.

## 11. Transaction boundaries



Use a transaction for logically related writes, such as creating an execution with initial tasks; saving a measurement and evidence links; saving candidate assessment and gate outcomes; saving a report and its claims; or recording a human review. Do not hold a transaction open during external requests or model calls. Persist validated outcomes in a short transaction. If a write fails, do not leave inconsistent partial results.



## 12. Repository boundaries



Suggested responsibilities (create only when the workflow needs them):



- `ExplorerQueryRepository`: bounded Explorer query lifecycle and retention metadata.
- `FrameManifestRepository`: versioned manifests and frame membership/order.
- `ObservationFrameRepository`: normalized image-plane/frame metadata.
- `ArtifactRepository`: controlled artifact metadata and access-state references.
- `SourceSnapshotRepository`: immutable Explorer selection snapshots and append-only validations.
- `IdempotencyRepository`: request-key/hash uniqueness and replay metadata.
- `InvestigationRepository`: investigation creation and permitted metadata updates.

- `ExecutionRepository`: run creation and execution history.

- `TaskRepository`: task records, transitions, dependency status.
- `TaskAttemptRepository`: immutable attempt history and retry diagnostics.

- `DataRetrievalRepository`: source query parameters/outcomes.

- `ObservationRepository`: normalized source observations and product references.

- `MeasurementRepository`: immutable method-versioned measurements.

- `QualityCheckRepository`: quality outcomes and diagnostics.

- `EvidenceRepository`: evidence ledger and typed links.

- `CandidateRepository`: candidates and versioned assessments.

- `CatalogueMatchRepository`: catalogue outcomes and associations.

- `AgentRunRepository`: bounded agent invocation metadata and validated outputs.

- `ReportRepository`: report versions and evidence-linked claims.
- `ImageComparisonRepository`: conditional comparison operations and exact input-frame membership.
- `TargetResolutionRepository`: optional named-target resolution provenance.
- `ExternalOverlayRepository`: optional separately attributed catalogue overlay query provenance.

- `HumanReviewRepository`: append-only review decisions.



Do not duplicate persistence logic in API routes, agents, and services.



## 13. Testing requirements



### Schema/migration tests



- Alembic builds a clean PostgreSQL database and reports the expected revision.

- PostgreSQL foreign-key enforcement is active; invalid references fail.

- Unique constraints reject duplicate run numbers, task keys, assessment versions, and report versions.

- Invalid enum values and null required fields are rejected.

- Numeric validation rejects invalid uncertainty/coordinates.

- Rollback leaves no partial logical operation; concurrent requests cannot allocate duplicate run numbers or idempotency keys.

- Secrets are not persisted.



### Lifecycle/workflow tests



- Allowed task transitions pass and invalid transitions fail.

- Retry attempts are bounded; cancellation is not success.

- Unsupported requests yield `NOT_SUPPORTED`; inadequate inputs yield `INSUFFICIENT_DATA`.

- Successful empty retrieval differs from retrieval failure.

- Failed mandatory dependencies block dependent tasks.

- Re-analysis preserves earlier executions and measurements.



### Scientific-integrity tests



- Measurements include method version and input references.

- Unavailable uncertainty is not silently stored as zero.

- LLM output cannot overwrite measurement records.

- Report claims reference evidence.

- Catalogue no-match never becomes a confirmed-new-planet result.

- Synthetic data are clearly labelled.

- Missing provenance remains explicit.

- A changed method version creates a distinguishable result.
- Standalone Explorer queries and frames persist without requiring an investigation execution.
- Explorer browsing does not create investigation/execution/agent/candidate records.
- Frame manifests order compatible frames by actual observation time and preserve channel/band/array as separate dimensions.
- Explorer handoff creates an immutable source snapshot; stale/invalid snapshot validation blocks the dependent run.
- Artifacts are delivered only through controlled artifact IDs; private storage paths are not exposed.
- Same idempotency key/same request replays the original result; same key/different request is rejected.
- Task retries preserve separate immutable attempt records.



### Integration tests



Test the first supported workflow from a bounded Explorer query or labelled fixture through frame-manifest persistence, explicit handoff, immutable snapshot validation, deterministic analysis, evidence persistence, candidate assessment if applicable, and report generation. Include a direct-investigation path if it remains supported. Do not make normal tests depend on unverified live archive access; mark external integration tests separately. Run database integration tests against PostgreSQL.



## 14. Implementation order

Follow `IMPLEMENTATION_PLAN.md` and `MASTER_DEVELOPMENT_PROMPT.md`, with PostgreSQL as the final database:

1. Reconcile this document with all authoritative project specifications and synchronize older SQLite references in API, architecture, backend, PRD, and operations documents.
2. Confirm PostgreSQL deployment configuration, secret handling, runtime/migration roles, and CI test service.
3. Define the minimum P0 tables for the independent Explorer path and first investigation-to-report workflow.
4. Implement centralized SQLAlchemy 2.x PostgreSQL session/pool management and Alembic configuration.
5. Implement the first schema migration for the selected P0 entities, constraints, and indexes.
6. Implement repositories and Pydantic/API mapping without exposing ORM models.
7. Implement bounded Explorer query, observation/frame, manifest, artifact, and source-snapshot persistence.
8. Implement investigation/execution/task/task-attempt and idempotency persistence.
9. Implement measurements, quality checks, evidence links, and provenance.
10. Add candidate assessment, catalogue/target-resolution, optional comparisons/spectra/overlays, report claims, and human review only as corresponding capabilities become ready.
11. Test constraints, concurrency, transactions, migration/rollback, backup/restore, and all source-snapshot integrity rules on PostgreSQL.
12. Verify provenance, versioning, report-claim integrity, artifact access, and historical preservation before release.

Do not implement the full optional target schema in one large migration. Do not postpone core provenance or Explorer handoff integrity until after report generation exists. Do not retain SQLite as a production fallback.

## 15. Decisions to record before implementation



Record in `IMPLEMENTATION_STATUS.md` or the decision log:



- Exact investigation/execution/retrieval/report/agent-run statuses from the authoritative workflow/API documents.

- Whether authentication/user ownership is in MVP scope.

- The first supported data product and fields it actually supplies.

- Minimum entities required for the first end-to-end workflow.

- Whether `report_claim_evidence` is required in the first migration for strict FK integrity.

- Whether candidate/assessment tables are in the first slice or deferred.

- Artifact storage backend, backup/PITR, retention, and deletion policy.
- PostgreSQL server version, hosting model, TLS, connection-pool, timeout, and migration-role settings.
- Redis usage, if any, and its non-durable role.
- Explorer-query retention and anonymous-session abuse-control policy.
- Frame-manifest retention/version policy and source-snapshot revalidation rules.

- ID format if already specified by API contracts.

- Timestamp and scientific numeric precision policy.



The database engine decision is already resolved: PostgreSQL is final. If a remaining decision affects scientific validity, data integrity, API contracts, or workflow semantics, do not guess silently; record and resolve the blocker before implementing that portion.



## 16. Definition of done



The database layer is complete for the MVP only when:



- Schema matches reconciled source-of-truth documents.

- PostgreSQL migrations apply to a clean database and supported upgrade path.

- PostgreSQL constraints, indexes, foreign keys, JSONB validation, and concurrency behavior are verified.

- Repository boundaries are respected.

- Transactions and lifecycle transitions are tested.

- Standalone Explorer query/frame/artifact records and investigation source snapshots are traceable.
- Required source data, measurements, quality checks, and evidence are traceable for the selected workflow.

- Historical runs and results are preserved.

- Reports are assembled from persisted evidence.

- Secrets and large science binaries are not stored in ordinary rows.

- PostgreSQL backup/restore/PITR and artifact backup/restore instructions match the implementation.

- Test outcomes are reported honestly as `PASS`, `FAIL`, `NOT RUN`, or `BLOCKED`.



## 17. Canonical document and repository ownership

- The canonical schema specification lives at repository root `database/DATABASE_SCHEMA.md`.
- Runtime SQLAlchemy sessions, ORM models, repositories, and database utilities live under `backend/src/helios_x/database/`.
- Alembic configuration and migration history live under `backend/migrations/`.
- `data/` contains runtime source files, cache, and temporary artifacts only; it is not a substitute for PostgreSQL and must not contain a committed local SQLite database.
- Do not maintain a second independently editable schema copy under `docs/`.
- API schemas remain Pydantic contracts; ORM models remain internal persistence models. Neither agents nor frontend clients query PostgreSQL directly.

### 17.1 Cross-document synchronization required by the final database decision

This schema supersedes previous draft statements naming SQLite as the initial database. Before implementation, update the database-platform references in `SYSTEM_ARCHITECTURE.md`, `API_SPECIFICATION.md` (descriptive database note only; API contracts remain database-independent), `BACKEND.md`, `PRD.md`, `OPERATIONS.md`, `IMPLEMENTATION_PLAN.md`, and `MASTER_DEVELOPMENT_PROMPT.md` to say PostgreSQL. Remove SQLite-only instructions such as SQLite foreign-key pragmas, local `.db` files, temporary SQLite migration tests, and SQLite backup procedures. Keep Redis optional and artifact storage separate from PostgreSQL. This synchronization must not alter the already-defined Explorer/investigation separation, scientific rules, or eight-agent boundary.

---

**Final rule:** HELIOS-X's PostgreSQL database must preserve what was observed, what was calculated, how it was calculated, which Explorer frames were selected for an investigation, what evidence supports a claim, and what remains unknown. Agents access approved service/repository tools; they do not receive unrestricted SQL credentials or bypass the application data-access layer. The database must never manufacture scientific certainty or erase history required to reproduce a result.
