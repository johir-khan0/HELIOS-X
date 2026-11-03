
# HELIOS-X — System Architecture

**Document:** `SYSTEM_ARCHITECTURE.md`  
**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Version:** 2.0 — Integrated SPHEREx Explorer + Agentic Investigation Architecture  
**Status:** Proposed canonical architecture baseline  
**Architecture style:** Modular monolith with separately buildable frontend applications  
**Primary mission:** NASA SPHEREx  
**Primary archive:** NASA/IPAC Infrared Science Archive (IRSA)  
**Frontend stack:** React, TypeScript, Vite  
**Backend stack:** Python, FastAPI, Pydantic  
**Initial persistence:** PostgreSQL, SQLAlchemy 2.x, Alembic  

> This document defines the intended architecture. It does not claim that source integrations, image products, scientific algorithms, agent workflows, consoles, or tests have already been implemented or validated. Implementation status must be reported separately and backed by tests or live verification.

---

## 1. Architecture Purpose and Product Identity

HELIOS-X is a public-facing SPHEREx sky exploration platform with an integrated agentic scientific investigation system. It has two connected capabilities that share a validated data foundation but have distinct user journeys.

1. **SPHEREx Sky Image Explorer — the primary public experience and NASA challenge deliverable.** A visitor enters Right Ascension (RA) and Declination (Dec), selects a supported field of view (FOV), discovers the actual observations available for that sky region, and views compatible image cutouts in acquisition-time order. The viewer provides frame navigation and video-like playback so users can visually inspect how the region changes over time.
2. **Agentic Scientific Investigation Engine — the deeper analysis capability.** A user explicitly promotes a selected Explorer region, observation sequence, or source into an investigation. The system validates data and prerequisites, runs supported deterministic scientific methods, coordinates specialized AI-agent roles, evaluates relevant alternative explanations, preserves evidence and provenance, prioritizes candidates for human review, and generates traceable reports.

The Explorer must work independently. A visitor must be able to search and view available images without creating an investigation, logging in, or invoking an AI agent. The investigation engine is not a prerequisite for image retrieval or playback. Conversely, the investigation engine may accept a target or region from other supported inputs where defined by the product requirements; it is not limited to Explorer-originated requests.

The product model is therefore:

**Explore the sky → notice a possible change → optionally investigate → review evidence → decide what deserves further scientific attention.**

HELIOS-X is not an automated planet-discovery authority. It can expose changes, calculate supported measurements, compare available catalogues, test alternatives, and prioritize candidates for review. It must not convert visual movement, an unmatched catalogue result, or an AI-generated interpretation into a claim that a new planet has been discovered.

## 2. Governing Architecture Decisions and Conflict Resolutions

The following decisions reconcile the NASA challenge with the earlier agentic-investigation plan. These decisions are architectural baselines for the dependent documents.

### 2.1 Explorer and Agentic Engine are equal platform capabilities, but not equal product entry points

**Conflict:** The earlier project positioned HELIOS-X primarily as an Agentic Scientific Investigation System. NASA requires a public-facing tool for exploring SPHEREx images over time.

**Resolution:** HELIOS-X retains both capabilities. The Sky Image Explorer is the default public entry point and the NASA-facing deliverable. The Agentic Investigation Engine remains a core capability and differentiator, accessed explicitly after or alongside exploration. The Explorer has its own acceptance gate and must work without the investigation engine.

### 2.2 Image playback and scientific motion measurement are separate capabilities

**Conflict:** A time-ordered visual comparison can reveal apparent changes, while a scientific motion claim requires registration, valid timestamps, source association, uncertainty, and quality checks.

**Resolution:** Basic image playback, timeline navigation, and blinking are Explorer functions. Source detection, astrometric registration, motion estimation, photometry, and scientific interpretation belong to the deterministic Scientific Engine and investigation workflow. Playback alone is never reported as a measured motion result.

### 2.3 React replaces Streamlit as the canonical frontend

**Conflict:** The earlier architecture proposed Streamlit for a rapid MVP, with React as a possible later replacement. The updated PRD and Backend specification define React, TypeScript, and Vite, and the NASA Explorer requires a purpose-built interactive image viewer and separately managed consoles.

**Resolution:** Use React + TypeScript + Vite for the Public Explorer, User Console, and Admin Console. Streamlit is not part of the canonical architecture. Do not maintain parallel Streamlit and React implementations of the same user experience. A temporary internal notebook or one-off research script is not a product frontend.

### 2.4 Public exploration and private investigation have different access boundaries

**Conflict:** The NASA-facing Explorer should be easy for anyone to access, while saved investigations, reviewer decisions, and administrative operations require authorization.

**Resolution:** Public, read-only Explorer operations may be unauthenticated, subject to strict server-side resource limits. The User Console owns authenticated user workflows and private resources. The Admin Console is a restricted operational interface and is never exposed as a public capability. Authentication and authorization are enforced by the backend, not by hiding frontend controls.

### 2.5 Root-level folders are separate product and technical boundaries

**Conflict:** The earlier structure primarily grouped most code under `frontend/` and `backend/`, while the updated project requires separate root-level areas for the Explorer, User Console, Admin Console, Agents, and database assets.

**Resolution:** Use the root-level monorepo structure defined in Section 10. Each frontend surface and the agent package has a separate root folder. The backend remains one modular-monolith application. The root `database/` folder holds canonical schema documentation and controlled database assets; the backend owns runtime persistence code, SQLAlchemy sessions, repositories, and Alembic migrations. This prevents two competing database implementations.

### 2.6 Agents are separate in the repository, not separate microservices in the MVP

**Conflict:** The project needs a dedicated `agents/` root folder, but it also requires a simple modular-monolith MVP.

**Resolution:** Store agent role implementations, prompts, schemas, and provider adapters in the root `agents/` package. The backend invokes them through typed interfaces in the same application runtime or through a bounded in-process adapter for the MVP. Do not deploy eight agents as eight services, require eight model instances, or give agents independent ownership of persistence and task state. A separate process or service may be introduced only after a measured requirement justifies it.

### 2.7 Mission cadence does not guarantee region-level coverage

**Conflict:** The NASA brief describes an approximately six-month all-sky mapping cycle, but a particular coordinate may not have a complete or compatible observation at each nominal cycle.

**Resolution:** Use actual archive products and validated acquisition timestamps. Return the true number of usable observations: zero, one, or multiple. Show gaps and unavailable observations; never synthesize frames to create a regular time series.

### 2.8 Spectral channels, detector arrays, and observation epochs are different dimensions

**Conflict:** SPHEREx has 102 spectral channels, while the challenge asks users to see how a sky region changes over time and mentions combining six detector arrays as a possible enhancement.

**Resolution:** Model observation epoch, spectral channel/wavelength, and detector-array identity as distinct metadata dimensions. Build a time sequence from actual acquisition times, not from channel number. Keep detector-specific data distinct unless documented product semantics and validated calibration justify combining them.

### 2.9 Optional NASA considerations must not delay the required Explorer

Image addition/subtraction, detector-array combination, on-the-fly spectral extraction, and Gaia/WISE overlays are valuable extensions, but they are conditional on suitable products, data semantics, method validation, and tests. Coordinate input, supported FOV selection, actual observation discovery, time-ordered image display, playback, and useful frame metadata belong to the core Explorer path.

### 2.10 Planet X research is a possible scientific use case, not the system's discovery claim

HELIOS-X may support investigation of unusual or unmatched candidates, including research questions sometimes associated with Planet X. It must not present every moving, variable, or unmatched source as a planet candidate. A possible candidate must pass applicable data-quality, registration, measurement, catalogue, alternative-hypothesis, and evidence gates. Candidate prioritization means “eligible for human review,” not “discovery” or “confirmation.”

---

## 3. Architecture Principles

1. **Explorer first, investigation optional:** the public image-viewing journey must not depend on agents or an investigation record.
2. **One shared data foundation:** retrieval, product metadata, image artifacts, quality information, and provenance should be reused by the Explorer and investigation engine.
3. **Modular monolith first:** keep one FastAPI backend application with clear internal modules; do not introduce microservices, distributed queues, or orchestration infrastructure without evidence of need.
4. **Separately buildable client surfaces:** Public Explorer, User Console, and Admin Console are separate root-level frontend applications sharing documented API contracts and reusable design primitives where appropriate.
5. **Deterministic science:** numerical measurements, image registration, source detection, image arithmetic, motion estimates, photometry, spectra, uncertainty, and statistical tests must come from versioned computational functions, never from LLM-generated values.
6. **Bounded agent autonomy:** agents may plan, request approved tools, interpret structured results, compare hypotheses, and recommend next steps. They cannot own authoritative task state, bypass gates, or overwrite observations and measurements.
7. **Evidence-based reasoning:** quantitative and scientific claims must link to the actual source product, observation, method, parameters, software version, result, quality outcomes, and uncertainty.
8. **Complete provenance:** preserve source archive, product and release identifiers, observation identifiers and timestamps, retrieval query, artifact references, checksums where computed, configuration, and method versions.
9. **Capability-aware execution:** only run a method when the selected data product, metadata, observation count, time baseline, uncertainty, and validated implementation support it.
10. **Explicit status semantics:** distinguish successful empty results, unavailable products, failed queries, corrupt products, unsupported operations, insufficient data, blocked work, and inconclusive outcomes.
11. **Scientific humility:** a visible image change is an apparent change until quality and registration are established; a catalogue no-match is not proof of novelty; candidate ranking is not discovery.
12. **Human scientific oversight:** significant candidate conclusions require human review and appropriate independent verification.
13. **Public/private separation:** anonymous users can only access specifically designated public read-only operations and public artifacts. Private investigations and administration require backend-enforced authorization.
14. **Artifact-first image delivery:** large FITS files and pixel arrays belong in controlled file/object storage, not relational database rows or ordinary JSON responses.
15. **No fabricated completeness:** missing epochs, unavailable channels, absent metadata, and unsupported methods must remain visible rather than being filled with synthetic-looking data.
16. **Reproducibility and history:** re-running an analysis creates a distinguishable execution/result version; it does not silently replace prior scientific evidence.
17. **Simple operational baseline:** PostgreSQL is the selected relational database for the MVP. Keep the backend as a modular monolith and introduce distributed workers or an object storage service only when operational requirements justify them.
18. **Contract-first integration:** the API specification owns HTTP contracts, the database specification owns persistence, methodology owns scientific method rules, and workflow specifications own task order and gates.

---

## 4. System Context and Logical Architecture

### 4.1 System context

HELIOS-X interacts with four user groups/surfaces:

- **Public visitor:** browses the Sky Image Explorer without an account; can inspect publicly available observations and frame metadata.
- **Registered researcher/user:** saves or manages investigations, starts supported runs, examines evidence, and retrieves reports through the User Console.
- **Reviewer/scientific lead:** inspects a candidate and its evidence, records a review decision, and requests further work where authorized.
- **Administrator/operator:** manages approved application configuration and operational visibility through the restricted Admin Console. Administrative actions do not alter scientific measurements or bypass quality gates.

External systems include the IRSA SPHEREx archive, supported auxiliary catalogues such as Gaia or WISE when integrated, and a replaceable LLM provider used only for bounded agent reasoning and language generation.

### 4.2 High-level logical architecture

```text
                                  USERS
                                    |
          +-------------------------+--------------------------+
          |                         |                          |
          v                         v                          v
  PUBLIC EXPLORER              USER CONSOLE               ADMIN CONSOLE
  React / TypeScript           React / TypeScript         React / TypeScript
  - RA / Dec input             - Saved investigations     - Operational status
  - FOV selection              - Run monitoring           - Source capabilities
  - Observation timeline       - Evidence / candidates    - Restricted controls
  - Image playback             - Reports / review         - Audit visibility
  - Frame metadata             - User preferences         - Admin-only actions
          |                         |                          |
          +-------------------------+--------------------------+
                                    |
                                    v
                         FASTAPI APPLICATION / API
                                    |
                  +-----------------+------------------+
                  |                                    |
                  v                                    v
          EXPLORER APPLICATION                  INVESTIGATION APPLICATION
          SERVICES                              SERVICES
          - Sky query service                   - Investigation service
          - Observation discovery               - Execution / task service
          - Frame manifest                      - Workflow orchestrator
          - Cutout / preview service             - Evidence / provenance
          - Artifact delivery                    - Candidate assessment
          - Cache / limits                       - Report builder
                  |                                    |
                  +-----------------+------------------+
                                    |
                                    v
                      SHARED SCIENTIFIC DATA FOUNDATION
          +-------------------------+--------------------------+
          |                         |                          |
          v                         v                          v
  DATA SOURCE ADAPTERS       DETERMINISTIC SCIENCE       METADATA / QUALITY
  - IRSA / SPHEREx           - Coordinates / WCS         - Product validation
  - Fixture adapter          - Image registration       - Quality flags
  - Gaia / WISE (optional)    - Cutout generation        - Calibration / PSF
                              - Source detection          - Uncertainty
                              - Change / motion           - Contamination
                              - Photometry / spectra
                              - Catalogue matching
          |                         |                          |
          +-------------------------+--------------------------+
                                    |
                                    v
                    CONTROLLED ARTIFACT + PERSISTENCE LAYERS
             +--------------------+-----------------------+
             |                    |                       |
             v                    v                       v
       FILE / OBJECT          POSTGRESQL / ORM       CACHE / RUNTIME
       STORAGE                - Investigations       - Derived previews
       - Original products    - Observations         - Frame manifests
       - FITS files           - Retrievals           - Query cache
       - Cutouts/previews     - Tasks / runs          - Temporary files
       - Derived products     - Measurements
                              - Evidence / candidates
                              - Reports / reviews
                                    |
                                    v
                         AGENT RUNTIME (8 LOGICAL ROLES)
                         Called through approved interfaces
                         - Planner / Retrieval / Quality
                         - Change / Motion / Spectral-Photometry
                         - Catalogue / Validation-Hypothesis
                                    |
                                    v
                          HUMAN REVIEW / REPORT OUTPUT
```

The diagram is logical, not a deployment diagram. In the MVP the FastAPI backend, internal services, workflow orchestrator, deterministic scientific modules, and agent runtime can run as one modular application. The three frontend applications are separate build targets and may be deployed independently or served under one domain. A single backend API remains the system contract boundary.

### 4.3 Architectural planes

1. **Presentation plane:** Public Explorer, User Console, and Admin Console.
2. **API/application plane:** FastAPI routes, authentication, request validation, services, and resource authorization.
3. **Data-access plane:** source adapters, archive clients, catalogue clients, and fixture readers.
4. **Scientific plane:** deterministic image processing, astronomy calculations, quality validators, and measurement algorithms.
5. **Agent/reasoning plane:** eight logical agents invoked through typed tools and constrained evidence access.
6. **Persistence/evidence plane:** PostgreSQL relational state, controlled artifact storage, cache, provenance, and report records.
7. **Operations/security plane:** configuration, logging, tracing, limits, audit records, backup, and deployment controls.

---

## 5. Product Surfaces and Access Boundaries

### 5.1 Public Explorer — `frontend/`

**Purpose:** deliver the NASA-required public-facing SPHEREx image exploration experience.

Core capabilities:

- RA/Dec input with clear units, coordinate format guidance, validation, and optional common target-name resolution if a verified resolver is integrated.
- FOV selection with supported bounds and explicit angular units.
- Optional time range, product release, channel/wavelength, and other supported filters.
- Actual archive observation discovery and coverage summary.
- Time-ordered image-frame manifest based on actual acquisition timestamps.
- Frame viewer with play, pause, previous, next, timeline scrubbing, playback speed, and visible timestamp.
- Blink/side-by-side comparison of selected compatible frames.
- Frame metadata and quality/contamination information when supplied by the source product.
- Explicit “Investigate this region/source” handoff that carries stable observation and query references, not client-calculated scientific measurements.
- Shareable URL parameters for supported coordinate/FOV state when implemented.

The Explorer does not create private investigation records merely to play images. It does not invoke the agent runtime for simple data retrieval or playback.

### 5.2 User Console — `user-console/`

**Purpose:** provide the authenticated research workspace for users who need saved work and deeper analysis.

Core responsibilities:

- Create, view, and manage owned investigations.
- Import a region/observation selection from the Public Explorer.
- Start, monitor, cancel, or retry eligible investigation executions according to authorization and API policy.
- Inspect task states, observations, measurements, quality checks, evidence, candidate assessments, and reports.
- Review provenance and scientific limitations.
- Manage user preferences and saved items supported by the backend.

The User Console is the application surface for the PRD's “investigation workspace.” It must not duplicate the Explorer's archive retrieval logic. It calls the same backend Explorer and investigation APIs.

A user may still enter an investigation target directly in the User Console, where the feature is supported. Explorer-to-investigation handoff is a convenient path, not the only valid scientific input path.

### 5.3 Admin Console — `admin-console/`

**Purpose:** provide restricted operational and administrative capabilities. It is separate from the public Explorer and ordinary user workspace.

Possible responsibilities, subject to approved product and deployment requirements:

- Inspect application health, readiness, job/task failures, and source-adapter status.
- Inspect supported source products, release configuration, and capability registry.
- Manage approved operational configuration through validated backend controls.
- Review audit records, access-control events, and system-level diagnostics.
- Manage users, roles, and administrative permissions if account management is enabled.
- View cache/storage usage and operational limits.

Admin Console must not:

- Expose raw SQL, shell, arbitrary file paths, unrestricted archive proxying, or direct ORM access.
- Permit an administrator or agent to rewrite scientific measurements or bypass scientific gates.
- Expose private investigation details without the correct authorization.
- Be publicly accessible without strong authentication and backend role checks.

**Delivery boundary:** the root-level folder and architectural boundary are defined now. Full Admin Console functionality is not a NASA P0 deliverable and should be built only when operational requirements and corresponding PRD/API/security contracts are approved. Do not create fake administrative controls merely to fill the UI.

### 5.4 Shared capabilities and help

Capabilities, product release information, supported channels, limitations, coordinate input guidance, mission attribution, and usage help can appear in the Public Explorer and User Console as contextually relevant. The backend capability registry is authoritative; frontends must not hard-code unsupported capabilities as available.

### 5.5 Access matrix

| Resource / action | Public visitor | Authenticated user | Reviewer | Administrator |
|---|---:|---:|---:|---:|
| Public sky query | Allow, rate-limited | Allow | Allow | Allow |
| Public image frame/artifact | Allow through controlled delivery | Allow | Allow | Allow |
| Public capability/help metadata | Allow | Allow | Allow | Allow |
| Create private investigation | Deny or explicitly limited guest flow only if approved | Own resources | As assigned | Only where operational policy permits |
| Read private investigation | Deny | Own/authorized | Assigned/authorized | Only with explicit audited permission |
| Cancel/retry execution | Deny | Own/authorized | If assigned role permits | Restricted operational policy |
| Record candidate review | Deny | Only if reviewer permission exists | Allow for assigned resources | Only with explicit reviewer/admin policy |
| Change source/system configuration | Deny | Deny | Deny | Allow through validated audited actions |

The exact roles and ownership rules are owned by `SECURITY.md`, PRD, and API contracts. The table is the architectural boundary, not proof that identity infrastructure exists.

---

## 6. Core Components and Responsibilities

### 6.1 Frontend applications

All product frontends use React, TypeScript, and Vite. Shared presentational primitives may live in `shared/` or an approved package, but each application owns its routes, feature code, and application state.

**Public Explorer** owns display and interaction: image rendering, timeline controls, frame navigation, URL state, and user-triggered region promotion. It does not perform authoritative scientific calculations.

**User Console** owns the authenticated investigation experience: form inputs, execution monitor, evidence navigation, review views, and report presentation. It displays authoritative values returned by the backend.

**Admin Console** owns operational presentation and validated administrative actions. The backend remains the authority for permissions and configuration changes.

No frontend may infer task success from a natural-language response, fabricate progress percentages, generate scientific values, or treat a preview image as the source product.

### 6.2 Backend API — FastAPI

Responsibilities:

- Provide a versioned REST/JSON API, initially under `/api/v1`.
- Validate coordinate, FOV, time, filter, ID, pagination, and action parameters with Pydantic.
- Apply authentication, authorization, public-operation allowlists, and rate/resource limits.
- Route Explorer operations to Explorer services and investigation operations to investigation services.
- Return stable response schemas, status values, and safe errors.
- Provide controlled image artifact delivery without exposing filesystem paths.
- Generate OpenAPI documentation from the implemented contract.

Route handlers should validate input, call an application service, and serialize a response. They must not contain FITS parsing, image registration, LLM prompts, scientific algorithms, persistence queries, or orchestration loops.

### 6.3 Explorer application services

The Explorer service group is responsible for:

- Sky-region query validation and canonicalization.
- Observation discovery and source-availability checks.
- Product/release selection under explicit policy.
- Frame manifest assembly and time ordering.
- Cutout/preview requests and controlled artifact references.
- Frame metadata retrieval.
- Comparison-product requests where supported.
- Caching and request-cost/resource controls.

A sky query is a read/discovery operation, not automatically an investigation. It must not create agent tasks, candidate records, or report records unless the user explicitly starts an investigation.

### 6.4 Investigation Service

Responsibilities:

- Create and manage investigation records.
- Validate the investigation objective, target, source observations, and requested analyses.
- Link the investigation to exact Explorer query, frame, observation, and product references when initiated from the Explorer.
- Create an execution record and request an approved workflow from the orchestrator.
- Expose investigation status and associated tasks/evidence/reports.
- Support controlled re-analysis without silently replacing prior results.

The service owns investigation-level business logic but delegates task lifecycle to the orchestrator and numerical work to the Scientific Engine.

### 6.5 Workflow Orchestrator

The Workflow Orchestrator is a deterministic system component, not an AI agent.

Responsibilities:

- Validate a proposed plan against capabilities, prerequisites, policy, resource limits, and allowed workflow definitions.
- Build and execute a dependency-aware task graph.
- Own task state transitions, retries, timeouts, cancellation, idempotency, and failure propagation.
- Persist task inputs, outputs/references, warnings, errors, attempts, and execution history.
- Enforce mandatory data-quality and evidence gates.
- Allow independent tasks to continue only when their prerequisites remain valid.
- Stop dependent tasks when mandatory prerequisites fail.
- Resume eligible tasks only when the persisted execution model supports it.
- Trigger candidate assessment and report generation only from persisted validated outputs.

The orchestrator must not let an LLM decide whether a mandatory gate passed or write authoritative task state directly.

### 6.6 Agent Runtime — root-level `agents/`

HELIOS-X retains the eight logical agent roles:

1. Investigation Planner Agent.
2. Data Retrieval Agent.
3. Data Quality Agent.
4. Change Detection Agent.
5. Motion Analysis Agent.
6. Spectral & Photometry Agent.
7. Catalogue Matching Agent.
8. Validation & Hypothesis Agent.

These are responsibilities, not eight required processes or model instances. Their detailed responsibilities, tools, schemas, and failure rules are owned by `AGENT_SPECIFICATIONS.md`.

The Agent Runtime must:

- Implement a common typed invocation interface and role registry.
- Receive validated task context and authorized evidence references.
- Use only allowlisted tools with validated arguments.
- Return schema-validated results, evidence references, warnings, and explicit failure states.
- Record model/provider and prompt-template versions where applicable.
- Use bounded timeouts and retries.
- Distinguish interpretation and hypothesis from source observations and computed measurements.
- Degrade safely when an LLM provider is unavailable.

Agents must never fabricate timestamps, archive records, source IDs, measurements, uncertainties, catalogue matches, or product availability; alter raw evidence; bypass capability/evidence gates; or make an automatic discovery claim.

### 6.7 Deterministic Scientific Engine

The Scientific Engine contains versioned, tested computational functions. Implement only methods justified by the selected data product and investigation objective.

Potential modules include:

- Coordinate conversion, validation, and angular separation.
- FITS/image reading and product validation.
- WCS inspection and coordinate-to-pixel transforms.
- Image cutout generation.
- Image registration and alignment-quality assessment.
- Source detection and centroid/position measurement.
- Epoch-to-epoch source association.
- Image difference generation where compatible.
- Motion estimation and uncertainty propagation.
- Photometry, SNR, and variability analysis.
- Spectral extraction/comparison and uncertainty-aware fitting.
- Catalogue cross-matching and match-ambiguity assessment.
- Statistical tests and multiple-testing controls where relevant.
- Artifact/contamination and alternative-hypothesis checks.
- Deterministic candidate-assessment inputs/metrics.

Every function must define supported inputs, units, coordinate/time conventions, method version, assumptions, uncertainty treatment, quality requirements, numerical acceptance criteria, and explicit failure behavior. No LLM is required to calculate a scientific result.

### 6.8 Data-Source Adapters

Every external archive or catalogue is accessed through a replaceable adapter interface.

**Primary adapter:** NASA/IPAC IRSA SPHEREx.  
**Development adapter:** local fixtures using the same normalized contract.  
**Optional auxiliary adapters:** Gaia, WISE, or other catalogues only after their access route, data rights, product semantics, and matching requirements are verified.

Adapters discover and retrieve data; they do not decide scientific meaning. They preserve actual source identifiers, release, acquisition time, metadata, query parameters, retrieval time, quality flags, units, WCS, and errors. They must distinguish successful empty results from failures and unavailable products.

### 6.9 Image Artifact Service and Storage

Responsibilities:

- Manage references to original archive products and derived artifacts.
- Generate or retrieve region cutouts and display previews where supported.
- Keep original scientific products separate from display derivatives.
- Store artifacts outside relational database rows.
- Validate file content, size, type, dimensions, WCS, and expected metadata before publishing a frame.
- Deliver images through controlled identifiers or approved streaming/temporary URLs.
- Apply authorization, size limits, caching, content-type rules, and artifact expiry where applicable.
- Preserve checksums and derivation provenance where computed.

The frontend must never receive absolute server paths or arbitrary file access. A JPEG/PNG preview may be used for display but must not replace the source FITS or other authoritative product for scientific processing.

### 6.10 Metadata and Quality Services

Normalize and validate available source metadata, including:

- Mission, instrument, product, release, processing version.
- Observation identifier and actual acquisition timestamp/time convention.
- Sky coordinates, coordinate frame/epoch where relevant, WCS, image size, pixel scale.
- Spectral channel/wavelength and detector-array identity where supplied.
- Units, calibration state, masks, flags, variance, PSF and contamination information when supplied.
- Retrieval timestamp, query, source references, and checksum when computed.

Unknown metadata must remain unknown or be marked unavailable. Do not invent default values that appear scientifically authoritative.

### 6.11 Persistence Layer

**Initial technology:** PostgreSQL + SQLAlchemy 2.x + Alembic + psycopg.

The backend owns runtime database code: connection/session management, ORM models, repositories, and migrations. The root `database/` folder contains the canonical schema document and any explicitly approved database-level assets; it is not a second runtime persistence implementation.

The relational store persists investigation state, execution/task records, retrieval metadata, observation references, measurement records, quality checks, evidence/provenance, candidate assessments, reports, human reviews, and references to image artifacts. Large FITS files, image arrays, previews, and spectral arrays remain in controlled file/object storage.

### 6.12 Evidence and Provenance Services

Evidence services organize the scientific record across Explorer-originated and direct investigations.

They must support:

- Source observation/product and retrieval references.
- Parent/input evidence relationships.
- Computed measurement records, units, uncertainties, method and software versions.
- Quality-check and evidence-gate outcomes.
- Distinction between observed facts, computed results, agent interpretations, hypotheses, and human decisions.
- Supporting, contradictory, and unresolved evidence.
- Provenance lookup from report claim to source product and method.
- Historical version preservation.

The public Explorer may show basic provenance/metadata without exposing private investigation details. A private evidence ledger is created only when an investigation or explicitly persisted analysis requires it.

### 6.13 Candidate Assessment Service

Candidate assessment is a deterministic service, not an AI agent.

It applies documented, versioned criteria to validated evidence and relevant gate outcomes. It must explain the factors that support or weaken a candidate, preserve contradictory evidence and limitations, and distinguish “prioritized for review” from scientific confirmation. Candidate status must not be based only on an LLM's confidence or narrative.

### 6.14 Scientific Report Builder

The report builder assembles reports from persisted validated records. An LLM may improve readability only when constrained to those records; all quantitative claims must resolve to stored computations or identified source data.

Reports should identify the investigation objective, data source/release, observation times, methods and versions, measurements/units/uncertainties, quality and gate outcomes, catalogue matches, alternative explanations, missing evidence, candidate status/rationale, warnings, and human-review state.

### 6.15 Human Review Service

Human review is a governed workflow step for candidates requiring expert attention.

Authorized reviewers may inspect evidence, accept a candidate for further investigation, reject it with a reason, defer a decision, or request re-analysis. Each decision records reviewer identity where applicable, timestamp, rationale, reviewed evidence/result version, and the resulting workflow action. Review actions never rewrite raw observations or scientific measurements.

### 6.16 Capability Registry

A backend-owned registry reports which sources, product releases, image operations, and scientific methods are available and under which prerequisites. The registry distinguishes:

- implemented and validated;
- implemented but not validated for the selected input;
- fixture-only;
- live-source unverified;
- unavailable;
- not supported;
- insufficient data for the current request.

Frontends and agents consume the registry; they must not independently assume capability availability.

---

## 7. End-to-End Data Flows

### 7.1 Public Explorer flow — independent of investigations

```text
Visitor opens Public Explorer
        |
        v
Enter RA / Dec + FOV (+ optional supported filters)
        |
        v
Frontend validates basic input and requests sky query
        |
        v
FastAPI public Explorer route
        |
        v
Sky Query Service
        |
        v
IRSA/SPHEREx Adapter (or clearly labelled fixture adapter)
        |
        v
Discover actual observations and products for the requested region
        |
        v
Validate product metadata, release, timestamps, WCS and quality fields
        |
        v
Generate frame manifest + controlled cutout/preview artifact references
        |
        v
Frontend displays actual time-ordered frames and frame metadata
        |
        v
User plays / pauses / scrubs / steps / blinks between compatible frames
        |
        +------> Continue public exploration
        |
        +------> Explicitly select "Investigate this region/source"
```

Important behavior:

- The query can return zero, one, or many actual frames.
- Zero frames is a valid empty-result state, not proof that no astronomical object exists.
- One frame is displayable but cannot establish temporal change by itself.
- Multiple frames are ordered by validated acquisition timestamp, not product filename or spectral channel index.
- The backend provides frame metadata and artifacts; the frontend controls basic playback.
- The request is not an investigation and does not invoke the agent runtime unless the user explicitly begins an investigation.

### 7.2 Explorer-to-Investigation handoff

```text
User selects region / frame sequence / detected source
        |
        v
Explicit "Investigate" action
        |
        v
Frontend sends selected query + stable observation/product references
        |
        v
Investigation Service revalidates all references and user permissions
        |
        v
Create investigation + execution records
        |
        v
Workflow Orchestrator checks capabilities and prerequisites
        |
        v
Retrieve/confirm required source products and preserve provenance
        |
        v
Deterministic scientific tasks execute where supported
        |
        v
Agents interpret validated outputs and propose permitted follow-up checks
        |
        v
Quality checks + alternative explanations + evidence gates
        |
        v
Candidate Assessment (if applicable) + evidence-linked report
        |
        v
Human review and recorded decision
```

The backend must not trust client-supplied measurements, timestamps, quality flags, WCS, or candidate status. It accepts identifiers and validated request parameters, then resolves authoritative source data and metadata server-side.

### 7.3 Direct investigation flow

A researcher may create an investigation from a known target identifier, coordinate, sky region, or supported dataset through the User Console. The Investigation Service uses the same adapters, scientific engine, orchestration, agents, evidence, and reporting modules. A direct investigation must still pass all capability and data validation gates.

### 7.4 Admin operational flow

An administrator enters the Admin Console, authenticates, and requests a permitted operational view or configuration action. The backend performs role and resource authorization, validates the action, records an audit event, and applies only allowed changes. Administrative controls do not provide a bypass into scientific methods or candidate decisions.

---

## 8. Sky Coordinates, Observation Time, Spectral Channels, and Detector Arrays

### 8.1 Coordinate conventions

- Accept RA and Dec in explicitly documented formats; normalize to a canonical internal representation such as decimal degrees.
- Document coordinate frame, epoch/reference metadata, units, and any transformations.
- RA must be finite and normalized to the supported angular range; wrap behavior must be explicit.
- Dec must be finite and within the valid celestial range.
- Reject ambiguous coordinate units or unsupported frames rather than silently guessing.
- Keep the user's original input for auditability alongside normalized values.
- If target-name resolution is implemented, preserve resolver name, release, coordinate result, uncertainty/source reference, and resolution status.

### 8.2 Observation epochs and timestamps

The Explorer frame sequence must use actual observation/acquisition timestamps from the selected product. Preserve the time scale, format, timezone convention, and timestamp source. Do not substitute retrieval time, processing time, release time, or filename order for acquisition time.

A six-month all-sky survey description is not a promise of exactly six-month regional cadence. Missing periods must appear as coverage gaps, not interpolated frames. A time-series scientific method must verify that the observation count and temporal baseline are suitable for that method.

### 8.3 Spectral channels

SPHEREx's 102 channels represent spectral samples across wavelength. They are not 102 separate observations in time. The system may support channel selection and spectral comparison, but channel changes must not be placed on the temporal playback axis as if they were epochs.

Frame sequences should be grouped or filtered so that frames are scientifically comparable—for example, by compatible channel/wavelength, product type, calibration state, and processing release. If a viewer supports a spectral dimension, it must be presented as a separate control/dimension.

### 8.4 Detector arrays

Detector-array identity must be retained where the product exposes it. The six-array combination described by the NASA challenge is an optional processing capability, not a default assumption. Combine detector-array products only when the official product semantics, calibration, PSF, spatial alignment, units, masks, and uncertainty rules support the operation. Preserve per-array provenance and make combined-product derivation reproducible.

### 8.5 Field of view

The API validates FOV geometry and supported maximum/minimum limits. The Explorer displays units and makes any limit visible. The backend should prefer an appropriately bounded cutout/query rather than downloading an unnecessarily large region. The FOV control must not imply uniform angular resolution or coverage beyond the selected product's capabilities.

---

## 9. Image Retrieval, Frame Manifest, and Viewer Architecture

### 9.1 Retrieval pipeline

1. Validate/canonicalize coordinates, FOV, time filters, channels, and release filters.
2. Query the configured SPHEREx/IRSA access route using documented parameters.
3. Record the exact query and retrieval attempt.
4. Normalize returned observation/product identifiers and metadata.
5. Validate product availability, release, dimensions, units, timestamps, WCS, quality fields, and required file structure.
6. Determine whether the products can support the requested cutout and viewer operation.
7. Retrieve or generate a WCS-aware cutout/display derivative when supported.
8. Store the original product and derived artifacts in controlled storage with provenance.
9. Return a frame manifest with stable IDs, actual timestamps, metadata, and controlled artifact references.
10. Allow the frontend to render frames in validated chronological order.

### 9.2 Frame manifest

The API contract should define a versioned frame manifest. Each frame should expose, as available:

- stable frame, observation, product, and artifact identifiers;
- mission, instrument, product type, release, and processing version;
- actual acquisition timestamp and its time convention;
- spectral channel/wavelength and detector-array identity as separate fields;
- image dimensions, pixel scale, units, and WCS metadata/reference;
- quality, contamination, mask, variance, PSF, and calibration indicators/references;
- original product reference and display artifact reference;
- retrieval status, warnings, and checksum if computed;
- provenance and source documentation reference.

Unknown or missing metadata must be represented as unavailable/unknown, never as a fabricated default.

### 9.3 Frontend playback

The frontend owns basic playback controls:

- play/pause;
- previous/next frame;
- timeline scrubber;
- configurable playback interval;
- active-frame acquisition timestamp;
- frame counter and coverage gaps;
- optional blink mode between selected frames;
- optional side-by-side mode where screen size supports it.

A server-generated video is not required for the core viewer. Individual frame navigation must remain possible so users can inspect exact metadata for each observation. The client must not silently skip failed frames or change the sequence order.

### 9.4 Image registration and visual comparison

The viewer may provide a basic visual sequence without claiming astrometric alignment. When a registered/blink comparison is offered, the backend must apply a documented WCS/astrometric registration method and return registration quality information. If registration fails, show the original frames and a warning; do not imply a physical shift from unregistered images.

A displayed overlay or difference product must identify which inputs were compared, their timestamps, compatible channels/products, registration status, scaling, masking, and method version.

### 9.5 Preview versus scientific source

Display previews may be JPEG/PNG/WebP or another appropriate browser format, but they are derivatives. Scientific processing must use authoritative source data or a documented lossless/validated intermediate. Image stretch, normalization, resampling, color mapping, and scaling must be clearly identified as display transformations and not measurement operations.

---

## 10. Canonical Root-Level Repository Structure

The repository must use the following top-level structure. This is the canonical target layout for the combined Explorer + Agentic Investigation platform. It is a planned repository structure, not a claim that all listed files already exist. Create modules when their phase begins; do not create large numbers of meaningless empty files.

```text
helios-x/
│
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
│   ├── public/
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
│   │       │   ├── router.py
│   │       │   ├── dependencies.py
│   │       │   ├── errors.py
│   │       │   └── v1/
│   │       │       ├── router.py
│   │       │       └── endpoints/
│   │       │           ├── health.py
│   │       │           ├── capabilities.py
│   │       │           ├── explorer.py
│   │       │           ├── image_artifacts.py
│   │       │           ├── investigations.py
│   │       │           ├── executions.py
│   │       │           ├── tasks.py
│   │       │           ├── observations.py
│   │       │           ├── evidence.py
│   │       │           ├── candidates.py
│   │       │           ├── reports.py
│   │       │           ├── reviews.py
│   │       │           └── admin.py
│   │       ├── core/
│   │       │   ├── config.py
│   │       │   ├── logging.py
│   │       │   ├── security.py
│   │       │   ├── permissions.py
│   │       │   └── exceptions.py
│   │       ├── schemas/
│   │       ├── domain/
│   │       ├── database/
│   │       │   ├── session.py
│   │       │   ├── base.py
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
│   │       │   ├── reviews/
│   │       │   └── administration/
│   │       ├── orchestration/
│   │       ├── science/
│   │       │   ├── coordinates/
│   │       │   ├── imaging/
│   │       │   ├── registration/
│   │       │   ├── source_detection/
│   │       │   ├── change_detection/
│   │       │   ├── motion/
│   │       │   ├── photometry/
│   │       │   ├── spectroscopy/
│   │       │   ├── crossmatch/
│   │       │   ├── quality/
│   │       │   ├── statistics/
│   │       │   └── candidate_assessment/
│   │       ├── data_sources/
│   │       │   ├── base.py
│   │       │   ├── registry.py
│   │       │   ├── capabilities.py
│   │       │   ├── fixtures/
│   │       │   ├── irsa_spherex/
│   │       │   └── catalogues/
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
├── agents/                            # Eight logical AI-agent implementations
│   ├── pyproject.toml                 # optional separate package metadata
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
├── database/                          # Root-level database documentation/assets
│   ├── DATABASE_SCHEMA.md
│   ├── seeds/                         # only approved reference/demo seed manifests
│   └── sql/                           # only justified reviewed SQL scripts
│
├── shared/                            # Stable contracts shared across boundaries
│   ├── contracts/
│   │   ├── investigation.schema.json
│   │   ├── task.schema.json
│   │   ├── observation.schema.json
│   │   ├── frame-manifest.schema.json
│   │   ├── evidence.schema.json
│   │   ├── candidate.schema.json
│   │   └── report.schema.json
│   ├── fixtures/
│   └── generated-types/               # generated from the API contract where practical
│
├── tests/                             # Cross-application/system tests
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
├── docs/                              # Project specifications and decisions
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
├── data/                              # ignored runtime data; never serve as a public static root
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

### 10.1 Canonical layout rules

1. `frontend/`, `user-console/`, `admin-console/`, `backend/`, `agents/`, `database/`, and `shared/` are distinct root-level areas.
2. `frontend/` means the public Sky Image Explorer, not a generic parent containing all three applications.
3. `user-console/` is the authenticated research workspace; it implements the PRD investigation-workspace surface.
4. `admin-console/` is a restricted operations/admin application. Its existence in the target architecture does not make all admin functions an MVP/NASA requirement.
5. `backend/` is one FastAPI application. Do not create separate backend services for Explorer, agents, or the consoles in the MVP.
6. `agents/` contains the agent package. Do not duplicate the same agent source tree under `backend/src/helios_x/agents/`.
7. Root `database/` contains the canonical schema document and approved database assets. Runtime session/models/repositories stay in `backend/src/helios_x/database/`; Alembic migrations stay in `backend/migrations/`.
8. `database/DATABASE_SCHEMA.md` is the single canonical database schema specification. Do not create or maintain a second independently editable `docs/DATABASE_SCHEMA.md`; other project documents must reference the canonical root-level path.
9. `shared/contracts/` contains stable cross-boundary machine-readable contracts. API request/response authority remains `API_SPECIFICATION.md` and the implemented OpenAPI contract. Prefer generating TypeScript client types from OpenAPI instead of manually duplicating API shapes.
10. `data/`, `artifacts/`, and `logs/` are runtime locations, ignored by Git except for appropriate `.gitkeep` or documented examples. They must not contain secrets or unapproved data.
11. Do not create empty placeholder modules for every possible feature. Create a directory or module when its responsibility is scheduled and defined.

### 10.2 Why this is not a microservices architecture

Separate root folders are repository and ownership boundaries, not a requirement for independent deployment. The three frontend applications can build separately, while backend modules and the agent runtime run together in one FastAPI deployment. The database remains a single initial PostgreSQL persistence system. This yields clear separation without introducing network calls, service discovery, distributed transactions, or unnecessary infrastructure.

---

## 11. Module Ownership and Dependency Rules

### 11.1 Dependency direction

```text
Frontend applications
        |
        v
Public API contracts / OpenAPI
        |
        v
API routes and request/response schemas
        |
        v
Application services
        |
        +------> Domain rules / typed contracts
        |
        +------> Workflow Orchestrator
        |              |
        |              +----> Agent Runtime (approved tasks only)
        |              |
        |              +----> Deterministic Scientific Engine
        |              |
        |              +----> Data-source / catalogue adapters
        |
        +------> Repositories / artifact storage
                       |
                       v
                 PostgreSQL + file storage
```

### 11.2 Rules

- Frontends call the backend API; they do not import backend or agent Python packages.
- API routes depend on application services, not on SQLAlchemy models or scientific implementation modules directly.
- Application services coordinate domain operations and repository interfaces.
- The orchestrator owns execution lifecycle and gate enforcement.
- Agents use typed tools/ports and return structured outputs; they do not access arbitrary repositories or filesystem paths.
- The Scientific Engine is independent of LLM availability.
- Data-source adapters do not import agent prompts or decide candidate meaning.
- Repositories own persistence queries; do not scatter SQLAlchemy queries through routes, agents, or scientific functions.
- Artifact storage owns file access and authorization checks; callers pass stable artifact IDs, not arbitrary paths.
- Candidate Assessment Service owns candidate scoring/status policy.
- Report Builder reads validated persisted results and must not invent missing values.
- Shared contracts must not become a dumping ground for business logic or duplicated implementations.

### 11.3 Avoiding circular dependencies

Define small interfaces at boundaries. For example, the Orchestrator invokes an `AgentExecutor` interface; the Agent Runtime implements it. The Agent Runtime receives approved tool interfaces rather than importing the FastAPI application. The Scientific Engine returns typed results and does not import agents. The backend application wires the implementations at startup.

---

## 12. Data and Scientific Processing Architecture

### 12.1 Product qualification before processing

Before an observation can be used, validate the applicable:

- mission, instrument, source archive, product name, release, and processing version;
- observation identifiers and actual acquisition times;
- coordinate frame, epoch/reference metadata, units, and WCS;
- image dimensions, scale, spectral channel/wavelength, detector-array identity;
- calibration, variance, masks, PSF, quality flags, and contamination metadata;
- file integrity, readability, structure, and expected product semantics;
- permitted use and relevant archive/product limitations.

### 12.2 Image compatibility

Before making a temporal comparison, verify that the input frames are compatible in the dimensions required by the requested operation. Relevant checks may include:

- actual distinct acquisition timestamps;
- compatible sky region, coordinate frame, WCS, and pixel scale;
- compatible product type, release, calibration state, and units;
- compatible spectral channel/wavelength selection;
- appropriate masks, variance, PSF, and quality metadata;
- valid image registration with measured residuals and uncertainty.

Do not subtract, stack, or claim a motion measurement from incompatible frames. If the comparison is not scientifically supported, the Explorer can still display individual frames with a clear warning; the Scientific Engine must return the appropriate unsupported/insufficient-data outcome for the requested calculation.

### 12.3 Image addition, subtraction, and stacking

These are optional capabilities, not automatic operations on every frame sequence.

- **Subtraction/difference:** require compatible products, valid alignment, compatible units/calibration, suitable masks/variance handling, and a documented normalization procedure.
- **Addition/stacking:** require a scientifically justified objective and appropriate alignment, weighting, scaling, PSF, masks, and uncertainty handling. A visually brighter combined image is not automatically more scientifically useful.
- **Detector-array combination:** keep each detector's provenance; combine only after verifying the released product's detector semantics and calibration requirements.
- Return the input frame references, transformation parameters, method version, quality diagnostics, and output artifact reference.

### 12.4 Source identification and association

A detected source in one frame must not automatically be equated with a source in another frame. Epoch association requires positional uncertainty, detection quality, blending/confusion checks, and a documented matching procedure. Ambiguous associations must remain ambiguous.

### 12.5 Apparent change versus physical motion

A source may appear to shift because of actual motion, parallax, image misregistration, WCS error, PSF changes, blending, source confusion, detector artifacts, or measurement noise. The system must:

1. validate timestamps, coordinate metadata, WCS, and image quality;
2. register frames using an appropriate method and assess residuals;
3. detect/associate the source across compatible epochs;
4. measure positions and propagate measurement plus registration uncertainty;
5. test relevant alternative explanations;
6. compare appropriate known-object catalogues and external evidence when available;
7. report the result with limitations and confidence/quality semantics defined by the scientific methodology.

A visible pixel shift is not automatically proper motion. A measured angular displacement is not automatically an orbit. A moving object is not automatically a planet.

### 12.6 Object identification policy

The system may classify or describe an object only to the level supported by data and verified catalogue evidence. Possible categories include known star, variable star, asteroid, comet, brown dwarf, galaxy, quasar, supernova/transient, artifact, blended source, or unknown/unmatched source, but the availability and confidence of each classification depend on data and matching sources.

For Planet X-related research:

- An unmatched source is not proof of a new object.
- Movement alone is not evidence that an object is a planet.
- Candidate prioritization is not scientific discovery.
- Do not infer orbital elements when the number of observations, temporal baseline, astrometric precision, or geometry is inadequate.
- A potential Planet X candidate requires relevant known-object checks, alternative-hypothesis evaluation, independent observations/verification, and qualified human review.
- Never expose an automated `CONFIRMED_PLANET` or equivalent discovery status.

The applicable detailed rules are owned by `SCIENTIFIC_METHODOLOGY.md`, `INVESTIGATION_WORKFLOW.md`, and `AGENT_SPECIFICATIONS.md`.

### 12.7 Spectral analysis

Spectral analysis uses validated spectral measurements and channel/wavelength metadata. Keep spectral axis separate from observation time. Preserve units, calibration, uncertainty, extraction method, and any detector/channel combination. On-the-fly spectrum generation is an enhancement only when the selected product and extraction method support it.

---

## 13. Eight Agents and System Services

### 13.1 Eight logical agents

| Agent | Responsibility boundary |
|---|---|
| Investigation Planner | Converts a validated request into a proposed bounded plan; cannot approve its own plan or override gates. |
| Data Retrieval | Coordinates approved retrieval tasks and interprets structured retrieval outcomes; the adapter performs actual retrieval. |
| Data Quality | Interprets deterministic quality outputs and identifies limitations; validators calculate the results. |
| Change Detection | Interprets validated source/change measurements; scientific functions calculate differences. |
| Motion Analysis | Interprets validated motion estimates, significance, and limitations; scientific functions calculate positions/motion. |
| Spectral & Photometry | Interprets validated photometry, variability, and spectral outputs; deterministic functions calculate them. |
| Catalogue Matching | Interprets known-catalogue matches and ambiguity; adapters and matching algorithms produce match records. |
| Validation & Hypothesis | Evaluates alternative explanations and evidence gaps using supplied results; cannot assert unsupported discoveries. |

### 13.2 System components that are not agents

- Workflow Orchestrator.
- Scientific Engine and validators.
- Data-source and catalogue adapters.
- Image Artifact Service.
- Candidate Assessment Service.
- Evidence/Provenance Services.
- Scientific Report Builder.
- Human Review Service.
- Capability Registry.

These responsibilities remain deterministic application/system components. Do not add artificial LLM calls for persistence, image rendering, calculations, score calculations, task-state changes, or report assembly.

### 13.3 Agent invocation boundary

Every agent invocation must identify the investigation/execution/task, agent role/version, schema version, allowed tools, input evidence IDs, validated parameters, configuration/policy versions, timeouts, and attempt number as appropriate. The result must be structured, validated, and evidence-referenced.

Agent-generated prose is interpretation, not a source record or measured fact. LLM/provider failure must not corrupt results or block non-agent scientific operations that can continue safely.

---

## 14. API and Contract Architecture

### 14.1 API style

- REST over HTTP with JSON request/response bodies.
- Versioned base path initially `/api/v1`.
- FastAPI-generated OpenAPI is the implemented contract reference.
- Pydantic request/response schemas are separate from SQLAlchemy ORM models.
- Stable resource identifiers, explicit error envelopes, and timezone-aware timestamps.
- No route should expose ORM table structure or filesystem paths.

Exact endpoint names, payloads, status codes, pagination, errors, authentication requirements, and idempotency rules are owned by `API_SPECIFICATION.md`. This architecture describes endpoint groups, not an alternate API contract.

### 14.2 Explorer API groups

The contract should provide supported operations for:

- coordinate/FOV validation and sky-region query;
- observation discovery and coverage response;
- time-ordered frame manifest retrieval;
- frame metadata retrieval;
- controlled image cutout/preview delivery;
- optional frame comparison/difference products where validated;
- capability and product-release discovery.

### 14.3 Investigation API groups

The contract should provide supported operations for:

- investigation creation and retrieval;
- execution creation, status, cancellation, and permitted retry;
- task status and dependency details;
- retrieval and observation metadata;
- measurements, quality outcomes, and evidence;
- candidate assessment and rationale;
- reports and supported exports;
- human review actions and history where enabled.

### 14.4 Admin API groups

Admin endpoints must be explicitly separated or protected by backend role checks. They may expose health/operations status, source registry configuration, capability configuration, audit data, and approved user/role operations. A public client must never gain access merely by guessing an endpoint path.

### 14.5 URL-addressable Explorer state

When supported, a shareable Explorer URL may encode normalized RA, Dec, FOV, and safe viewer filter state. It must not embed secrets, private investigation IDs, arbitrary file paths, or untrusted external URLs. The server must revalidate all parameters when the link is opened.

### 14.6 Long-running jobs

Archive retrieval, cutout generation, and scientific investigations may be long-running. Return a job/execution resource and let the frontend poll the documented status endpoint or use a later validated push mechanism. Do not hold one HTTP request open for the entire investigation. Basic frame playback should use already available frame/artifact resources and must not require a long-running agentic execution.

---

## 15. Persistence, Artifact Storage, and Caching

### 15.1 Relational persistence

PostgreSQL is the selected relational persistence target for local development, CI, staging, and production, accessed through SQLAlchemy 2.x and migrated with Alembic using the configured PostgreSQL driver. The database should persist the minimum entities required by the supported workflow, such as:

- investigations and executions;
- tasks and task dependencies;
- data retrieval records;
- observations and product references;
- frame manifest records or reproducible references where persistence is required;
- measurements and scientific analysis runs;
- quality checks and evidence gates;
- evidence records and evidence relationships;
- candidate assessments;
- agent run metadata;
- reports, report claims, and versions;
- human reviews and audit references.

The exact schema, fields, indexes, constraints, and retention rules are owned by `DATABASE_SCHEMA.md`. Do not copy or redefine the schema in this document.

### 15.2 Large data artifacts

Store original FITS products, image arrays, generated cutouts, previews, spectra, and substantial derived files outside relational rows. Persist artifact ID, URI/reference, content type, dimensions, provenance, checksums where computed, and storage status in the database.

Use a controlled storage adapter so local filesystem storage can be replaced by object storage later without changing scientific or API contracts.

### 15.3 Runtime data versus database design assets

- Root `database/`: canonical schema documentation and reviewed database assets.
- `backend/src/helios_x/database/`: runtime database sessions, models, repositories, and persistence helpers.
- `backend/migrations/`: Alembic migration history.
- `data/`: runtime source products, cutouts, previews, cache, and temporary files.

These directories have different purposes and must not be duplicated or conflated.

### 15.4 Caching

Cache only products/results with an explicit key and provenance, such as normalized query, region/FOV, release, channel, observation IDs, processing version, and artifact derivation parameters. Cache entries must be invalidated or versioned when source release, product, query parameters, or processing method changes. Do not treat cached content as current archive availability without checking the applicable policy.

### 15.5 Transactions and history

- Keep database transactions short.
- Never hold a transaction open across archive or LLM network calls.
- Enforce foreign keys and uniqueness rules defined in the schema.
- Preserve earlier analysis runs and human decisions.
- Reanalysis with changed inputs, release, parameters, or algorithm version creates a distinguishable run/result version.
- Back up relational records and referenced artifacts together or with a verifiable manifest.

---

## 16. Scientific Status, Quality, and Failure Semantics

Task execution status, data retrieval status, quality outcomes, evidence-gate outcomes, and candidate status are separate concepts. Use the canonical values in the authority documents; do not invent synonyms in this architecture.

### 16.1 Task statuses

`QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRYING`, `CANCELLED`, `BLOCKED`, `NOT_SUPPORTED`, `INSUFFICIENT_DATA`.

### 16.2 Quality outcomes

`PASSED`, `FAILED`, `INCONCLUSIVE`, `NOT_AVAILABLE`, `NOT_APPLICABLE`.

### 16.3 Evidence-gate outcomes

`PASSED`, `FAILED`, `UNKNOWN`, `NOT_APPLICABLE`.

### 16.4 Candidate statuses

Use the canonical candidate model and source document. Candidate labels must not imply confirmation. In particular, `PRIORITIZED_FOR_REVIEW` means that the candidate is eligible for human attention, not that it has been scientifically confirmed.

### 16.5 Explorer retrieval outcomes

Explorer responses must clearly distinguish at least:

- successful retrieval with multiple usable frames;
- successful retrieval with one usable frame;
- successful empty result (no matching observations returned);
- data/product unavailable for the requested region/filter;
- unsupported query/filter/capability;
- insufficient metadata or unusable product;
- provider timeout/failure;
- corrupt or invalid product;
- partial retrieval with warnings and explicit missing frames.

A successful empty result is not a failed query and is not proof that no object exists. A failed query is not an empty result. A partial result must not be described as complete.

### 16.6 Failure behavior

- Retry only transient failures and only within bounded policies.
- Do not retry invalid parameters or deterministic validation failures without changed input.
- Preserve attempt history and earlier failures.
- Cancellation must not be overwritten by late success.
- Unknown or inconclusive gates never silently pass.
- A task with no valid persisted output cannot be marked successful merely because an agent returned prose.
- If image retrieval fails, preserve the error and allow the Explorer to display a truthful error/partial state.
- If an agent fails, continue independent deterministic tasks only when safe and allowed.
- If report language generation fails, return validated structured records and mark the optional summary failure.

---

## 17. Security, Privacy, and Public Deployment

Follow `SECURITY.md`; this architecture adds product-specific boundaries.

### 17.1 Public Explorer

- Expose only explicitly designated read-only query, capability, frame-manifest, and controlled image-delivery operations anonymously.
- Apply strict RA/Dec, FOV, time-range, channel, result-count, file-size, timeout, concurrency, and rate limits.
- Never provide an unrestricted proxy to IRSA or arbitrary URLs.
- Validate source product references and artifact IDs before delivery.
- Prevent SSRF, path traversal, arbitrary file reads, and unbounded downloads.
- Use caching where policy permits, without losing release/product provenance.
- Use HTTPS and explicit CORS origins in public deployment.
- Keep public query telemetry free of secrets and unnecessary personal data.

### 17.2 User Console and private investigations

- Require authentication for private resources.
- Enforce ownership/role checks on every investigation, execution, evidence, candidate, report, retry, cancellation, and review request.
- Prevent a client from changing ownership or submitting fabricated measurement values.
- Use resource-level authorization; obscuring buttons is not security.

### 17.3 Admin Console

- Require strong authentication and explicit administrator roles.
- Separate read-only operational roles from configuration-changing roles where appropriate.
- Audit every consequential configuration or access-control action.
- Do not expose unrestricted database, shell, filesystem, model prompt, or source-archive access.
- Keep privileged endpoints unavailable to public users even if the Admin Console bundle is discovered.

### 17.4 Agents and external data

- Treat FITS headers, archive metadata, catalogue text, file names, and external content as untrusted data, never instructions.
- Give agents only typed, allowlisted tools and minimum necessary evidence.
- Never let agents issue arbitrary shell, Python, SQL, or filesystem operations.
- Store provider credentials only in backend environment/secrets; never in frontend bundles or reports.
- Validate file content and resource limits before processing.

### 17.5 Data integrity and privacy

- Preserve source files and raw observations as immutable inputs.
- Do not store credentials or secrets in relational records.
- Apply documented retention rules to user investigations, logs, cache, and generated artifacts.
- Record human-review identity and decision metadata under the approved privacy policy.

---

## 18. Performance, Reliability, and Scalability

### 18.1 MVP baseline

Start with:

- one FastAPI modular-monolith backend;
- one PostgreSQL database for initial single-instance development;
- controlled local artifact storage behind an interface;
- three separately buildable React applications;
- an in-process bounded workflow runner;
- fixture data for deterministic development tests;
- one verified live SPHEREx/IRSA path before claiming live support.

### 18.2 Performance principles

- Fetch the smallest useful region/product first; avoid unnecessary full-product downloads.
- Enforce FOV, frame-count, file-size, concurrency, and processing limits.
- Generate display previews/cutouts as controlled derived artifacts.
- Use pagination for large observation/candidate/report collections.
- Cache repeatable query and artifact operations with clear invalidation rules.
- Avoid loading full-resolution image arrays into browser memory when a viewport-sized preview is sufficient.
- Process expensive scientific tasks outside ordinary request handlers.
- Avoid repeatedly downloading identical archive products when a valid cached product can be reused.

### 18.3 Scaling triggers

Consider database capacity tuning/scaling, a durable job queue, worker processes, or object storage only when measured needs justify them—for example concurrent writes, multi-instance deployment, long-running jobs that must survive restarts, artifact volume, bandwidth, or throughput requirements. PostgreSQL is already the selected relational database; these decisions do not reopen the selected database platform. Any change must preserve API and scientific contracts.

Do not add Kafka, Celery, Redis, Kubernetes, service mesh, microservices, or a distributed agent topology merely for architectural appearance.

### 18.4 Degraded operation

- Explorer may remain usable for cached/publicly available products when the agent provider is unavailable.
- Agentic investigation must continue deterministic tasks that do not require the LLM where safe.
- If IRSA is unavailable, show cached data only with its provenance and freshness state, or clearly labelled fixtures in development/demo mode.
- If one optional catalogue is unavailable, record the limitation; do not imply that cross-matching completed.
- If one optional analysis is unsupported, allow independent supported outputs to remain available.

---

## 19. Observability, Auditability, and Operations

Record, as applicable:

- request/correlation ID;
- user/session or access class without logging secrets;
- investigation, execution, task, frame, observation, artifact IDs;
- source adapter, product release, query parameters, and retrieval status;
- method/agent/provider/prompt version;
- start/end timestamps, duration, and retry attempt;
- quality and evidence-gate outcomes;
- warnings, structured errors, cancellation, and state transitions;
- cache hit/miss and artifact delivery status.

Do not log full FITS payloads, image arrays, secrets, or large binary content. Health and readiness must distinguish application liveness, database readiness, artifact-store readiness, and external archive reachability. Basic application startup must not be misreported as proof of IRSA availability or scientific validity.

Operational instructions, configuration keys, backup/restore, retention, and deployment procedures are owned by `OPERATIONS.md`.

---

## 20. Testing and Architecture Acceptance

The architecture is acceptable for implementation only when the module boundaries and contracts are coherent. Product release acceptance is separately governed by `TESTING_AND_ACCEPTANCE.md` and the PRD.

### 20.1 Architecture-level tests

- Frontend apps use the documented API contracts and do not calculate authoritative scientific results.
- The public Explorer can query and display available frames without creating an investigation or calling an LLM.
- Explorer-to-investigation handoff passes stable identifiers and is revalidated by the backend.
- Agents cannot write authoritative task state, modify raw measurements, or bypass evidence gates.
- Data-source adapters can be replaced by fixture adapters without changing the scientific engine.
- Large image products are stored outside relational rows and delivered through controlled artifact access.
- Database migrations apply to a clean temporary PostgreSQL database.
- Public endpoints are rate/size/resource limited; private and admin resources require server-side authorization.
- A source failure, empty result, missing metadata, corrupt image, unsupported method, and insufficient-data outcome remain distinct.
- Re-analysis preserves historical runs and evidence.

### 20.2 Explorer acceptance gate

The Explorer architecture is ready for a supported release when:

1. A visitor can use the public Explorer without creating an investigation.
2. RA/Dec and FOV inputs are validated and normalized.
3. The backend discovers actual SPHEREx observations for the requested sky region.
4. The API returns a truthful zero/one/multiple-frame manifest.
5. Frames are ordered by actual validated acquisition timestamps.
6. Each available frame has a controlled image artifact and relevant metadata.
7. The UI supports play/pause, step, timeline navigation, and visible active-frame time.
8. Failed, partial, empty, and unsupported results are clearly represented.
9. Image comparisons do not imply physical motion unless scientific registration and measurement checks have passed.
10. The five NASA target scenarios are exercised against actual archive availability; unavailable coverage is reported honestly.

### 20.3 Agentic investigation acceptance gate

The investigation architecture is ready when:

1. A bounded investigation can be created directly or from Explorer context.
2. The orchestrator validates capabilities and task dependencies.
3. Deterministic scientific methods produce tested, structured outputs.
4. Agents use narrow contracts and approved tools only.
5. Data quality, uncertainty, alternative explanations, and evidence gates are retained.
6. Candidate assessment is deterministic and explainable.
7. Reports are assembled from persisted validated evidence.
8. Human review decisions are recorded against the relevant evidence version.
9. Agent/provider failure cannot invent or corrupt scientific results.

### 20.4 Integrated platform acceptance gate

The combined platform is ready when both independent paths work and the handoff works:

- Public Explorer works without an investigation.
- User Console can create and monitor an investigation.
- An Explorer selection can be promoted to an investigation without losing query/observation provenance.
- Results and reports trace to the exact source products, observation times, methods, parameters, and software versions.
- Admin Console functions, when enabled, are protected and audited.

### 20.5 NASA target validation matrix

The architecture and test suite must include dedicated scenarios for:

- Barnard's Star — target resolution, actual regional observation coverage, and suitability for a positional-change workflow.
- Epsilon Indi B — target/region resolution and the limits of resolving or distinguishing the brown-dwarf pair in the selected products.
- WISE 0855 — use the challenge-supplied coordinate reference (RA `08h55m10.831s`, Dec `-7d14m42.53s`) and verify coverage and actual temporal observations.
- RW Aurigae — target resolution, available observations, and suitability for a variability workflow.
- R Draconis — target resolution, available observations, and suitability for a variability workflow.

The presence of a target in this list does not guarantee that the archive provides the necessary number, quality, cadence, channel, or resolution of observations. Record actual archive results and limitations. Never fabricate a target frame to make a test pass.

---

## 21. Implementation Sequence and Phase Gates

Build the system in dependency order. The Public Explorer must not be postponed until after the entire agent platform; equally, the agentic plan must not be discarded. The recommended sequence builds the common data foundation and then completes both paths with explicit integration gates.

### Phase 0 — Repository and document reconciliation

- Review PRD, Architecture, Data Source, Scientific Methodology, Workflow, Agent Specifications, Implementation Plan, Database Schema, API Specification, UI/UX, Testing, Security, Operations, Backend, and Frontend documents in the controlling review order.
- Confirm the root-level structure in Section 10 and reconcile the implementation plan.
- Resolve the Streamlit-versus-React conflict in favor of React + TypeScript + Vite.
- Decide canonical file locations and avoid duplicate database schemas or duplicate agent implementations.

**Exit:** document conflict register is resolved or explicit blockers are recorded; target repository layout is agreed.

### Phase 1 — Repository and application foundations

- Create the root-level folders and package/workspace configuration.
- Initialize the three React app shells: Public Explorer, User Console, and Admin Console (Admin may remain a protected shell until approved features are scheduled).
- Initialize the FastAPI modular monolith, settings, logging, health/readiness, CORS policy, and pytest.
- Add shared contract generation/validation strategy.

**Exit:** clean repository setup runs the backend and the intended frontend development apps; no fake features are presented.

### Phase 2 — Contracts, database, and artifact storage foundation

- Finalize API schemas, shared IDs/statuses, coordinate/FOV/time conventions, frame-manifest shape, and error semantics.
- Finalize database schema and migrations for observations, artifacts, investigations, tasks, evidence, and provenance as required by the first workflows.
- Add controlled local artifact storage and reference-based delivery.

**Exit:** a clean PostgreSQL database migrates; typed metadata/artifact references can be persisted; large images are not stored in relational rows.

### Phase 3 — Fixture adapter and metadata validation

- Implement the fixture adapter with realistic, clearly labelled metadata fixtures.
- Validate coordinates, FOV, timestamps, WCS, units, channels, releases, and product structure.
- Distinguish empty query, failed query, unavailable data, corrupt product, and partial results.

**Exit:** deterministic no-network tests pass and no fixture is presented as live SPHEREx data.

### Phase 4 — Explorer retrieval and frame-manifest backend

- Implement sky-region query and observation-discovery services.
- Build chronologically ordered manifests from actual fixture timestamps.
- Implement controlled cutout/preview generation and delivery.
- Add cache keys, limits, and frame metadata endpoints.

**Exit:** fixture query returns correct zero/one/multiple frame states and each valid frame is viewable through the controlled API.

### Phase 5 — Public Sky Image Explorer

- Implement RA/Dec and FOV input, frame viewer, timestamp display, timeline, play/pause, frame stepping, and blink/side-by-side behavior as supported.
- Add URL-addressable coordinate state where contracted.
- Clearly label fixture data and unsupported features.

**Exit:** end-to-end fixture Explorer flow works without an investigation record or LLM call.

### Phase 6 — Live IRSA/SPHEREx integration and coverage verification

- Re-check current official archive and release documentation.
- Implement the documented access route behind the adapter interface.
- Validate product metadata, WCS, quality fields, timestamps, and retrieval provenance.
- Test the five NASA targets and record actual available coverage.

**Exit:** live support is claimed only for the exact products/operations verified; unavailable target coverage is transparently documented.

### Phase 7 — Deterministic scientific engine baseline

- Implement only data-supported image registration, source measurement, change/motion, photometry, spectral, and quality methods selected for the first investigations.
- Define numerical acceptance tolerances and known-answer fixtures.

**Exit:** every included scientific method passes deterministic tests; unsupported operations return explicit outcomes.

### Phase 8 — Investigation foundation and orchestration

- Implement investigation/execution/task services, task graph, dependency rules, gates, cancellation, retry, idempotency, and persisted history.
- Support direct target/region requests and Explorer-originated handoff.

**Exit:** one bounded fixture investigation can execute, persist results, and expose honest status.

### Phase 9 — Evidence, candidate assessment, reports, and human review

- Implement evidence/provenance relationships, versioned deterministic candidate policy, report builder, and authorized human review actions.

**Exit:** one investigation produces a traceable report; review decisions are auditable and do not alter measurements.

### Phase 10 — Agent integration

- Integrate the eight logical agent roles incrementally through typed interfaces and allowlisted tools.
- Begin with Planner, Retrieval, Data Quality, and Validation roles where useful; add analysis interpretation agents only when deterministic outputs exist.

**Exit:** agent outputs are schema validated and evidence linked; agent failure cannot corrupt scientific data or bypass gates.

### Phase 11 — User Console integration

- Implement authenticated investigation creation, monitoring, evidence/candidate inspection, reporting, and review views.
- Enforce resource ownership/roles through the backend.

**Exit:** the complete researcher workflow works without relying on mock backend data.

### Phase 12 — Admin Console and operational controls

- Implement only approved operational views/actions, access control, audit logging, capability/source visibility, and operational monitoring.

**Exit:** all enabled admin functions are protected, audited, and backed by real API behavior. Otherwise keep them unavailable or deferred.

### Phase 13 — Optional scientific and Explorer enhancements

- Add image addition/subtraction/stacking, detector-array combination, on-the-fly spectrum extraction, Gaia/WISE overlays, advanced filtering, or larger-scale processing only after data semantics and scientific tests are defined.

**Exit:** each optional capability is verified and supported or explicitly marked unavailable/not supported.

### Phase 14 — Integrated validation and release review

- Run Explorer, investigation, and integration acceptance suites.
- Verify scientific claims, public resource limits, private authorization, admin protections, migrations, backup/restore, artifact handling, and provenance.
- Reconcile documents against actual implementation status.

**Exit:** all applicable P0 gates pass; unverified items and limitations are disclosed.

---

## 22. Architecture Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Product becomes an agent dashboard instead of the NASA Explorer | Make independent public Explorer acceptance a release gate and the default entry point. |
| Explorer is coupled to an investigation run | Keep query/frame retrieval separate from investigation creation and agent orchestration. |
| Apparent image shift is mistaken for physical motion | Validate WCS, registration, timestamps, source association, quality, and uncertainty before motion claims. |
| A moving/unmatched source is called a new planet | Enforce catalogue, alternative-hypothesis, evidence-gate, and human-review policy; prohibit automatic discovery status. |
| Nominal six-month cadence is assumed for each region | Use actual per-region acquisition timestamps and display missing coverage. |
| 102 channels are treated as time epochs | Model spectral channel and observation time as separate dimensions and test this distinction. |
| Detector arrays or releases are combined incorrectly | Preserve identity/provenance and require validated product semantics before combination. |
| Large image data overload the relational database/API/browser | Use controlled artifact storage, cutouts, previews, size limits, and streaming/reference delivery. |
| Public endpoints are abused | Apply rate limits, FOV/frame caps, request timeouts, artifact controls, caching, and abuse monitoring. |
| Private investigations leak through public APIs | Enforce backend authentication, ownership, role checks, and resource-level authorization. |
| Admin controls bypass scientific policy | Restrict administrative actions to operational configuration; audit changes; never expose scientific gate overrides. |
| Agent reasoning fabricates data | Give agents structured evidence only, enforce schemas/tool allowlists, and validate all quantitative claims. |
| Duplicate root/database or agent implementations emerge | Use canonical paths and explicit directory ownership rules in Section 10. |
| Architecture becomes over-engineered | Keep one backend modular monolith, PostgreSQL, and bounded in-process execution until measurable needs justify scaling. |
| Documents drift from implementation | Use contract ownership, serial review, decision records, and phase acceptance gates. |

---

## 23. Source-of-Truth Documents and Contract Ownership

This architecture defines system boundaries and dependency direction. It does not replace detailed subsystem specifications.

| Document | Authority |
|---|---|
| `PRD.md` | Product identity, surfaces, users, requirements, priorities, user journeys, non-goals, and acceptance gates. |
| `SYSTEM_ARCHITECTURE.md` | System components, boundaries, data flows, root structure, deployment model, and dependency rules. |
| `DATA_SOURCE_SPEC.md` | SPHEREx/IRSA products, release policy, archive access, metadata, retrieval, target validation, and source limitations. |
| `SCIENTIFIC_METHODOLOGY.md` | Scientific assumptions, equations, registration, motion, photometry, spectra, uncertainty, quality, alternative hypotheses, and Planet X policy. |
| `INVESTIGATION_WORKFLOW.md` | Investigation task order, dependencies, gates, statuses, branches, and re-analysis behavior. |
| `AGENT_SPECIFICATIONS.md` | Eight agent roles, tool permissions, contracts, failure behavior, and acceptance tests. |
| `API_SPECIFICATION.md` | Exact HTTP routes, request/response schemas, error envelopes, statuses, pagination, authentication, and idempotency. |
| `database/DATABASE_SCHEMA.md` | Persistence entities, columns, relations, constraints, indexes, and migrations. |
| `BACKEND.md` | Backend implementation details for API services, Explorer, image processing, artifacts, orchestration, persistence, and runtime. |
| `FRONTEND.md` | React application structure, viewer components, user workspace, API integration, states, and frontend tests. |
| `UI_UX_SPECIFICATION.md` | Navigation, visual system, Explorer interactions, console workflows, responsive behavior, and accessibility. |
| `TESTING_AND_ACCEPTANCE.md` | Test cases, fixtures, scientific tolerances, NASA target scenarios, release blockers, and end-to-end acceptance. |
| `SECURITY.md` | Secrets, authentication, authorization, public API controls, admin roles, untrusted data, and audit security. |
| `OPERATIONS.md` | Setup, configuration, run commands, migrations, logging, artifact storage, backup, recovery, and deployment. |
| `IMPLEMENTATION_PLAN.md` | Repository creation, implementation sequence, milestones, dependencies, and phase gates. |
| `MASTER_DEVELOPMENT_PROMPT.md` | Required document-review order, conflict reconciliation, execution discipline, phase gates, and reporting rules. |

### 23.1 Conflict-resolution hierarchy

When documents disagree:

1. PRD controls what the product must do and what is in/out of scope.
2. System Architecture controls component boundaries and system-level interactions.
3. Data Source Specification plus verified official product documentation controls archive/product semantics and data availability.
4. Scientific Methodology controls scientific validity, measurements, uncertainty, alternative hypotheses, and interpretation.
5. Investigation Workflow controls scientific task sequence, dependencies, and evidence gates.
6. Agent Specifications control agent responsibilities and permitted behavior.
7. API Specification controls public HTTP behavior.
8. Database Schema controls persistence structure.
9. Backend and Frontend specifications control implementation details within those contracts.
10. Testing and Acceptance controls evidence required to claim a feature works.
11. Security and Operations control safe deployment and runtime behavior.
12. Implementation Plan controls build sequence but cannot override product, scientific, API, or data-source requirements.
13. Master Development Prompt coordinates the review and implementation process; it cannot silently redefine the source contracts.

Record each material conflict and its resolution in `docs/decisions/` before implementing the affected feature. Do not silently change a downstream file to conceal an upstream contradiction.

---

## 24. Open Decisions Before the Affected Feature Is Implemented

The following are deliberate verification points, not permission to invent assumptions:

1. Which exact SPHEREx products/releases and archive access route will support each initial Explorer target?
2. What are the actual region-specific observation times and coverage for all five NASA validation targets?
3. Which image products can be converted to browser-displayable cutouts without losing required metadata/provenance?
4. Which viewer library best supports astronomical image display, WCS-aware interaction, large images, and accessibility? Choose after a small technical proof of concept.
5. Which image registration algorithm and acceptance tolerances are justified for the chosen product?
6. Which image channels/products are comparable across epochs, and how should incompatible frames be grouped in the UI?
7. What detector-array metadata and combination semantics are available in the official product documentation?
8. Which auxiliary Gaia/WISE catalogue products and releases are appropriate for the target validation cases?
9. Which authentication provider/approach is required for the User and Admin Consoles?
10. What exact Admin Console actions are required for the initial deployment, beyond read-only operations visibility?
11. Will anonymous users be allowed to launch a limited guest investigation, or will all investigations require an account? Public image browsing remains unauthenticated.
12. What storage backend is used in deployment: local controlled filesystem initially, or object storage when scale/deployment requires it?
13. Which API operations require asynchronous jobs, and what durability/restart behavior is required?
14. Which optional features (image arithmetic, spectrum generation, detector combination, overlays) have verified inputs and acceptance criteria?

For each decision, record the selected option, rationale, verification evidence, owner, and remaining limitation in the relevant specification or decision record.

---

## 25. Final Architecture Rule

HELIOS-X must be built as one coherent platform with two independent but connected user journeys:

**Public Explorer:** coordinate input → actual SPHEREx observation discovery → validated image frames → time-based visual exploration.

**Agentic Investigation:** explicit investigation request → capability checks → deterministic science → bounded agents → alternative hypotheses and evidence gates → candidate assessment → report → human review.

**Integrated workflow:** a user can move from an Explorer region or source into an investigation without losing provenance, but browsing never depends on an investigation.

Keep the repository boundaries explicit, the backend modular, the agent runtime bounded, the database singular, the scientific methods deterministic, and the public/private access model enforceable. Preserve the previous agentic-investigation plan while making the NASA-required Sky Image Explorer the primary public experience. Do not remove a planned capability merely to simplify the diagram; defer optional capabilities through explicit phase gates when their data or validation requirements are not yet satisfied.

---

**End of System Architecture Document.**
