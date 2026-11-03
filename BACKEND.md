
# HELIOS-X Backend Engineering Specification

**File:** `BACKEND.md`  
**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Status:** Extended MVP implementation specification  
**Backend:** Python, FastAPI, Pydantic  
**Persistence:** PostgreSQL, SQLAlchemy 2.x, Alembic  
**Primary archive:** NASA/IPAC Infrared Science Archive (IRSA)  
**Primary mission:** SPHEREx  

> This document defines intended backend behavior. It does not claim that the backend, archive integration, image viewer, scientific methods, or tests are already implemented or validated.

---

## 1. Purpose and Product Boundary

The HELIOS-X backend is the trusted data, execution, scientific-computation, and evidence layer for two integrated product capabilities:

1. **Public SPHEREx Sky Image Explorer:** lets users enter a sky position or select a sky region, discover available SPHEREx observations, retrieve suitable image cutouts, and inspect the available observations as a time-ordered image sequence.
2. **Agentic Scientific Investigation Engine:** lets a user start a structured investigation from a sky region, selected observations, or a candidate; validates the available data; runs supported deterministic scientific analyses; coordinates bounded AI-agent tasks; preserves evidence and provenance; evaluates candidates using documented rules; and produces traceable reports for human review.

The Image Explorer is the primary public-facing entry point for the NASA SPHEREx challenge. The Agentic Investigation Engine is an integrated analysis layer, not a substitute for the Explorer. Users must be able to explore available images without running an AI investigation. They may explicitly promote a selected region or observation sequence into an investigation through a supported backend workflow.

The backend must:

- Accept and validate sky-region queries, including Right Ascension (RA), Declination (Dec), field of view (FOV), observation-time constraints, spectral-channel selection, and supported product-release constraints.
- Discover available SPHEREx observations from the configured archive adapter.
- Retrieve and validate image products and metadata, then produce controlled image cutouts or browser-displayable previews.
- Return a time-ordered manifest of available image frames so the frontend can provide timeline, playback, blinking, and frame-to-frame navigation.
- Preserve the original product, observation time, release, WCS, units, channel/wavelength, calibration metadata, quality flags, and source provenance.
- Provide scientifically constrained image comparison operations only when their input products are compatible.
- Accept an Explorer region or selected observation set as the starting context for an agentic investigation.
- Orchestrate the existing eight logical AI-agent roles while keeping numerical analysis in deterministic, versioned scientific code.
- Preserve evidence, uncertainty, failures, alternative explanations, and historical results.
- Expose stable, versioned REST/JSON contracts to the React frontend.

> HELIOS-X does not ask AI to guess what an anomaly is. It lets users inspect the observations, investigates measurable changes in validated data, tests alternative explanations, connects results to evidence, and explains what is known and unknown. Final scientific judgment remains with humans.

### 1.1 MVP goals

- Python + FastAPI + Pydantic.
- REST/JSON API at `/api/v1`.
- PostgreSQL persistence through SQLAlchemy 2.x and Alembic.
- A modular monolith, not premature microservices.
- A working coordinate-to-observation-to-image-frame Explorer flow.
- A working timeline/image-sequence contract for the frontend; playback itself is a frontend presentation behavior.
- A fixture adapter first; live IRSA/SPHEREx access only after product feasibility and exact archive access methods are verified.
- One reproducible, bounded end-to-end agentic investigation launched from supported Explorer context.
- Deterministic, tested image and scientific processing where included in the MVP.
- Traceable observations, image artifacts, measurements, evidence, uncertainty, quality outcomes, method versions, and reports.
- Frontend-visible retrieval state, frame availability, errors, unsupported operations, and insufficient-data outcomes.
- pytest unit, integration, scientific, adapter, API, and end-to-end tests.

### 1.2 Explicit non-goals

Do not start with Kubernetes, Kafka, distributed services, or a complex autonomous-agent framework. Do not claim Planet X or another new planet has been detected or confirmed. Do not treat catalogue no-match as proof of novelty. Do not treat SPHEREx's 102 spectral channels as 102 time epochs. Do not assume every sky region has multiple observations or that every requested channel/product is available. Do not let an LLM generate measurements, uncertainties, archive records, timestamps, image pixels, or catalogue matches. Do not create synthetic frames and present them as archive-derived images. Do not claim features are implemented merely because they are documented.

Do not make image playback depend on running the agentic pipeline. Do not require a user to create an investigation merely to view publicly available images. Do not implement scientifically invalid image addition, subtraction, stacking, or cross-release comparisons just to expose a viewer control.

---

## 2. Source-of-Truth Documents

Keep this file aligned with the project's source documents:

| Document | Authority |
|---|---|
| `PRD.md` | Product identity, goals, users, MVP scope, non-goals, and user journeys |
| `SYSTEM_ARCHITECTURE.md` | Component boundaries and system-level data flow |
| `AGENT_SPECIFICATIONS.md` | Agent roles, inputs, outputs, approved tools, and tool limits |
| `SCIENTIFIC_METHODOLOGY.md` | Scientific assumptions, algorithms, uncertainty, quality, and limitations |
| `INVESTIGATION_WORKFLOW.md` | Investigation order, task dependencies, and evidence gates |
| `IMPLEMENTATION_PLAN.md` | Build sequence and repository conventions |
| `DATA_SOURCE_SPEC.md` | IRSA/SPHEREx products, releases, metadata, and retrieval methods |
| `DATABASE_SCHEMA.md` | Tables, fields, relationships, constraints, indexes, and migrations |
| `API_SPECIFICATION.md` | Exact endpoints, payloads, status codes, pagination, and error envelope |
| `TESTING_AND_ACCEPTANCE.md` | Required tests and release blockers, including NASA challenge target tests |
| `SECURITY.md` | Security, public-access, permissions, and untrusted-data requirements |
| `OPERATIONS.md` | Setup, configuration, artifact storage, logs, backups, and recovery |
| `UI_UX_SPECIFICATION.md` | Frontend screens, viewer behavior, user interactions, and states |
| `FRONTEND.md` | Frontend implementation details and API integration |

### 2.1 Contract ownership and conflict rule

- `API_SPECIFICATION.md` controls exact public HTTP paths, methods, request/response fields, status codes, pagination, and error formats.
- `DATABASE_SCHEMA.md` controls persistence entities, fields, relationships, constraints, and migrations.
- `DATA_SOURCE_SPEC.md` and verified official product documentation control archive access, product semantics, releases, and source metadata.
- `SCIENTIFIC_METHODOLOGY.md` controls scientific methods, preconditions, uncertainty, quality rules, and interpretation limits.
- `INVESTIGATION_WORKFLOW.md` controls agentic task order, dependencies, and evidence gates.
- `UI_UX_SPECIFICATION.md` and `FRONTEND.md` control presentation and client interactions.
- `SECURITY.md` controls authentication, authorization, public access, and security requirements.

This file coordinates backend implementation; it must not silently redefine another document's contract. When an implementation requirement introduced here needs an API, database, UI, security, or testing change, record that dependency and update the authoritative document before implementing the affected behavior.

**Important current alignment dependency:** the existing API, database, UI/UX, and implementation documents must be extended to cover the Explorer capabilities described here. Until those contracts are updated, endpoint names and new persistence fields in this document are capability requirements, not permission to invent a competing public contract or database schema.

---

## 3. Technology Baseline

- Python, FastAPI, and Pydantic.
- PostgreSQL, SQLAlchemy 2.x, Alembic, and the selected PostgreSQL driver (`psycopg`).
- pytest.
- NumPy, SciPy, Astropy, and other astronomy libraries only where required by selected, verified scientific methods.
- FITS-aware image reading and WCS handling through validated libraries selected in the dependency review.
- React + TypeScript + Vite frontend calling the API over HTTP.
- Environment-backed configuration; `.env.example` contains placeholders only.
- FastAPI-generated OpenAPI.
- Provider-independent LLM interface, isolated from deterministic scientific computation.
- Local data/artifact storage for the initial MVP; object storage may be introduced when a real deployment requirement justifies it.
- Docker optional, not required for the local MVP.

Pin versions in the actual project configuration. The real repository configuration and lock files take precedence over example commands here. Do not add a large image-processing dependency solely because it appears in a proposal; select libraries after validating product formats and the required operations.

---

## 4. Integrated Backend Architecture

```text
React / TypeScript Public Web Client
        |
        | REST + JSON /api/v1
        v
FastAPI Routes + Pydantic Contracts
        |
        +--> Sky Explorer API / Application Services
        |       +--> Sky-region query validation
        |       +--> Observation discovery
        |       +--> Image/cutout retrieval
        |       +--> Frame manifest and artifact delivery
        |       +--> Optional compatible image comparison
        |       +--> Viewer metadata and quality information
        |
        +--> Investigation API / Application Services
        |       +--> Investigation / Execution / Task
        |       +--> Workflow Orchestrator
        |              +--> Investigation Planner Agent
        |              +--> Data Retrieval Agent
        |              +--> Data Quality Agent
        |              +--> Change Detection Agent
        |              +--> Motion Analysis Agent
        |              +--> Spectral & Photometry Agent
        |              +--> Catalogue Matching Agent
        |              +--> Validation & Hypothesis Agent
        |
        +--> Deterministic Scientific Engine
        |       +--> FITS/WCS validation and image preparation
        |       +--> Image registration and comparison, when supported
        |       +--> Source detection and measurement, when supported
        |       +--> Motion / photometry / spectral analysis, when supported
        |
        +--> Data-source adapters
        |       +--> Fixture adapter
        |       +--> IRSA/SPHEREx adapter
        |       +--> Optional Gaia / WISE catalogue adapters
        |
        +--> Evidence / Provenance services
        +--> Candidate Assessment Service (deterministic rules)
        +--> Scientific Report Builder
        |
        +--> SQLAlchemy repositories --> PostgreSQL
        +--> Controlled local data/artifact storage
```

Use a **modular monolith**. Internal components communicate through typed Python interfaces and application services. Not every agent, adapter, or module needs an HTTP endpoint.

### 4.1 Separate user journeys; shared scientific foundation

The backend must support two distinct but connected journeys:

**Explorer-only journey**

1. User enters RA/Dec or selects a supported sky region.
2. Backend validates the coordinates, FOV, time constraints, and supported query options.
3. Data-source adapter discovers observations and available products.
4. Backend retrieves the required image products or uses verified cached artifacts.
5. Backend validates the products and creates displayable cutouts/previews.
6. Backend returns a frame manifest with actual observation times and artifact references.
7. Frontend displays the returned frames in a timeline and provides play/pause/blink/navigation controls.

This journey must not create an investigation record or invoke agents merely to display images.

**Explorer-to-investigation journey**

1. User selects **Investigate this region** or an equivalent supported action.
2. Frontend submits the selected sky region, time range, relevant observation/frame identifiers, and requested analysis options.
3. Backend creates an investigation and execution through the documented investigation API.
4. Workflow Orchestrator validates product capability, temporal coverage, metadata, quality, and method prerequisites.
5. Deterministic scientific functions execute supported operations.
6. Agents interpret validated structured outputs and may recommend additional supported checks.
7. Evidence, provenance, task state, candidate assessment, and reports are persisted.
8. Frontend links the investigation results back to the relevant Explorer region and observation frames.

Do not pass large image arrays through the investigation-creation request. Pass stable observation/product/frame references and validated query parameters. Backend services retrieve the controlled artifacts when execution requires them.

### 4.2 Trust boundaries

- **API:** untrusted requests enter here; validate all inputs.
- **Explorer services:** enforce query limits, source availability checks, artifact authorization, and viewer-specific business rules.
- **Application services:** enforce business rules and coordinate transactions.
- **Orchestrator:** owns investigation task order, dependencies, statuses, retries, cancellation, and gate enforcement.
- **Agents:** coordinate, request approved tools, and explain structured outputs; they do not own authoritative task state.
- **Scientific Engine:** calculates measurements and image-derived metrics deterministically.
- **Adapters:** retrieve and normalize source data; they do not decide scientific meaning.
- **Artifact service:** controls storage and delivery of image files; it must not expose arbitrary filesystem paths.
- **Candidate Assessment Service:** applies documented/versioned criteria.
- **Report Builder:** uses persisted validated results; it must not add unsupported facts.

Route handlers should validate, call a service, and serialize a response. Do not put archive query logic, FITS parsing, scientific calculations, LLM prompts, orchestration loops, filesystem traversal, or report assembly inside routes.

---

## 5. Suggested Backend Structure

Align with `IMPLEMENTATION_PLAN.md` before creating files. The repository implementation plan places the application package under `backend/src/helios_x/` and uses `database/` for its persistence module. Use one canonical package root; do not maintain duplicate package roots or create both `db/` and `database/` implementations.

```text
backend/
  src/helios_x/
    main.py
    config/
      settings.py
    api/
      dependencies.py
      errors.py
      routes/
        health.py
        capabilities.py
        sky_regions.py              # Explorer region/coordinate queries
        image_frames.py             # Frame manifest and image artifact access
        image_comparisons.py        # Only if supported by the API contract
        investigations.py
        executions.py
        tasks.py
        data_retrievals.py
        observations.py
        evidence.py
        candidates.py
        reports.py
    schemas/
      explorer.py                   # Sky query, frame manifest, viewer metadata
      observations.py
      investigations.py
      evidence.py
    database/
      base.py
      session.py
      models/
      repositories/
    services/
      sky_explorer_service.py
      observation_service.py
      image_artifact_service.py
      image_comparison_service.py    # Only for validated comparison operations
      investigation_service.py
      execution_service.py
      evidence_service.py
      candidate_service.py
      report_service.py
    orchestration/
    agents/
    science/
      image_validation/
      image_registration/           # Add when validated method is selected
      image_comparison/              # Add when validated method is selected
      source_detection/              # Add when validated method is selected
      motion/                        # Add when validated method is selected
      photometry/                    # Add when validated method is selected
      spectroscopy/                  # Add when validated method is selected
    data_sources/
      base.py
      fixtures/
      irsa_spherex/
      catalogues/
        gaia/                        # Optional; only when integrated
        wise/                        # Optional; only when integrated
    evidence/
    candidates/
    reporting/
    llm/
alembic/
tests/
  fixtures/
  unit/
  integration/
  scientific/
  api/
  acceptance/
data/                               # ignored runtime data and artifacts
  cache/                             # ignored derived/cache artifacts if configured
logs/                                # ignored runtime logs
```

This is a logical target, not an instruction to create empty modules. Create a module when its interface and implementation are required by an approved phase. The exact package/module names must be reconciled with `IMPLEMENTATION_PLAN.md` and the repository before creation.

### 5.1 Directory responsibilities

- `api/`: HTTP routes, dependencies, response serialization, and error mapping.
- `schemas/`: Pydantic request/response and internal typed contracts.
- `database/`: ORM models, session, repositories, and persistence helpers.
- `services/`: Explorer, image artifact, observation, investigation, evidence, candidate, and report operations.
- `orchestration/`: task graph, state machine, retries, cancellation, and gates.
- `agents/`: eight logical agent roles with narrow contracts.
- `science/`: deterministic image and astronomical algorithms and validators.
- `data_sources/`: adapters, retrieval clients, normalized metadata, and fixtures.
- `evidence/`: lineage, provenance, quality, and evidence links.
- `candidates/`: deterministic versioned assessment criteria.
- `reporting/`: report assembly and serialization.
- `llm/`: optional provider-independent client and prompt templates; no scientific calculations.
- `config/`: validated application and archive settings.
- `data/`: controlled runtime storage for downloaded source products, derived cutouts/previews, and other artifacts. It must not be served as an unrestricted static directory.

Do not create empty modules just to make the tree look complete.

---

## 6. Local Setup and Configuration

### 6.1 Prerequisites

Install Git, a Python version supported by the repository configuration, and Node.js LTS for frontend integration. Docker is optional.

### 6.2 Virtual environment

Windows PowerShell:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

macOS/Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

Install dependencies using the committed project configuration. If `pyproject.toml` defines editable installation and development extras, the intended command may be:

```bash
pip install -e ".[dev]"
```

Only use it once that configuration exists. Do not maintain an unmanaged second dependency list.

### 6.3 Environment

Create `.env` from `.env.example` if the project uses dotenv. Proposed keys (the implementation must define and validate actual settings):

```dotenv
APP_ENV=development
API_HOST=127.0.0.1
API_PORT=8000
DATABASE_URL=postgresql+psycopg://<user>:<password>@<host>:5432/<database>
LOG_LEVEL=INFO
FRONTEND_ORIGIN=http://localhost:5173
DATA_DIR=./data
CACHE_DIR=./cache
REPORTS_DIR=./data/reports
MAX_SKY_QUERY_AREA=<validated application limit>
MAX_FRAME_COUNT=<validated application limit>
MAX_IMAGE_BYTES=<validated application limit>
ARCHIVE_CONNECT_TIMEOUT=<configured duration>
ARCHIVE_READ_TIMEOUT=<configured duration>
```

The angle/area, frame-count, artifact-size, timeout, and concurrency limits must be selected from actual product needs and deployment capacity. Do not treat the placeholders above as literal environment values or silently invent scientific limits.

Add provider credentials only when needed. Never commit `.env`, database dumps, logs, downloaded products, cached images, or generated reports. Keep runtime files outside the public frontend directory.

A common development launch command after the entrypoint exists is:

```bash
uvicorn helios_x.main:app --reload --host 127.0.0.1 --port 8000
```

This is an example, not proof the app currently starts. Align the module path and command with the chosen repository layout and `OPERATIONS.md`.

---

## 7. API and Frontend Alignment

`API_SPECIFICATION.md` is the sole authority for exact paths, methods, payloads, status codes, pagination, and error response structure. Do not create a second contract in frontend code or treat the illustrative operation groups below as final endpoint names.

### 7.1 Required API capability groups

The public API contract must eventually define the following capability groups:

**Explorer and sky-region access**

- Validate and normalize RA/Dec and coordinate frame/reference metadata.
- Search available observations for a sky position/region, FOV, time interval, product release, and optional spectral channel/filter criteria.
- Return actual matching observation records and retrieval/availability outcomes.
- Retrieve a frame manifest containing the ordered observations and associated image artifact metadata.
- Retrieve a controlled image cutout or browser-displayable image representation by a stable observation/product/artifact reference.
- Retrieve frame details and relevant product/quality metadata.
- Expose comparison operations only when their inputs satisfy documented compatibility rules.

**Investigation and evidence access**

- Health/readiness and capability discovery.
- Investigation creation and retrieval.
- Execution creation, status, cancellation, and supported retry behavior.
- Task status and execution details.
- Data retrieval, observation, measurement, and evidence retrieval.
- Candidate assessment and candidate details.
- Report retrieval and supported export operations.

The exact route names and request/response schemas must be finalized in `API_SPECIFICATION.md` before route implementation. Prefer stable resource identifiers and avoid making the frontend reconstruct archive filenames or internal disk paths.

### 7.2 Sky query contract requirements

The API specification should define a typed sky-region query with applicable fields such as:

- `ra_deg` and `dec_deg` in explicitly documented units.
- Coordinate frame and epoch/reference metadata where required by the source or method.
- FOV width/height or a documented supported region geometry, with explicit angular units.
- Optional observation start/end time using a documented time convention.
- Optional channel/wavelength selection.
- Optional product-release constraint, with server-side validation against supported releases.
- Optional maximum result/frame count only if the server enforces a safe cap.

Validation requirements:

- RA must be finite and within the normalized supported range; wrapping behavior must be documented.
- Dec must be finite and within the valid celestial range.
- FOV must be positive, finite, and inside configured server limits.
- Time ranges must be ordered and use explicit timezone/time-scale semantics.
- Channel and release values must be validated against the selected data product and archive documentation.
- The backend must not silently reinterpret coordinate units, frames, epochs, or time scales.
- A coordinate-only query may return zero, one, or multiple observations; do not fabricate a frame to fill a sequence.

### 7.3 Frame manifest contract requirements

For every available frame, the backend should return (as supported by the source product):

- Stable frame, observation, and product identifiers.
- Actual acquisition/observation timestamp and time convention.
- Mission, instrument, product name, release, and processing version.
- Selected spectral channel/wavelength and detector/array identity where the metadata provide them.
- Image dimensions, pixel scale, units, WCS reference or validated WCS metadata.
- Quality, contamination, mask, variance, PSF, and calibration references/flags where available.
- A controlled artifact reference or API URL for the cutout/preview.
- Original product reference, retrieval status, checksum if computed, and provenance.
- A clear indication of unavailable metadata rather than a fabricated default.

The frame manifest must distinguish observation time from retrieval time, product release time, and spectral-channel identity. Frames should be ordered by actual acquisition time, not by archive filename or spectral channel number.

### 7.4 Image artifact delivery

The backend must not put large FITS files or raw pixel arrays into ordinary JSON responses. Use controlled artifact delivery, such as a backend streaming endpoint or a short-lived controlled URL, according to the deployment and API contract.

Requirements:

- Never expose an absolute server filesystem path to the client.
- Validate that each requested artifact belongs to an authorized, known product/observation record.
- Restrict delivery to approved file types and generated preview variants.
- Enforce artifact size and request limits.
- Support appropriate content type, caching, and range behavior only where the selected delivery mechanism and client need it.
- Preserve the original FITS/product separately from any generated preview or cutout.
- Label previews as display derivatives; do not use a lossy preview as the authoritative scientific input.
- Never allow a user-provided artifact path to be opened directly by the server.

### 7.5 Long-running work

Archive discovery, product retrieval, cutout generation, and scientific investigation may outlast a browser request. For work that is not safely completed within a normal request, create/return a documented job or execution resource and let the frontend poll its status endpoint. Do not keep a single HTTP request open for the full agentic workflow or a large archive transfer.

An in-process MVP runner is acceptable only if its limitations are documented. Do not claim durable distributed execution or restart recovery unless implemented and tested. The Explorer should not require a long-running investigation execution merely to request an already available image frame.

### 7.6 Frontend behavior supported by backend contracts

The backend supplies observations, timestamps, frame manifests, image artifacts, quality metadata, and comparison results. The frontend owns presentation behavior such as:

- Play, pause, step forward/back, and timeline scrubbing.
- Automatic frame sequencing and playback speed controls.
- Blink comparison between selected frames.
- Display mode and panel layout.
- Coordinate parameters reflected in a shareable URL.

The backend may provide server-side frame ordering and validated comparison products, but it must not create a video solely to implement basic playback unless a separate requirement justifies it. The viewer must be able to navigate individual frames so that users can inspect exact observation metadata.

---

## 8. Explorer Image Retrieval and Processing Pipeline

### 8.1 Request path

A standard Explorer request follows this controlled path:

1. **Validate request:** validate coordinates, coordinate frame, FOV, time range, channel, release, and configured limits.
2. **Discover coverage:** ask the selected data-source adapter which relevant SPHEREx observations/products are actually available for the requested region.
3. **Normalize records:** convert only documented archive fields into the internal observation/product schema while retaining original source metadata needed for audit.
4. **Select products:** select compatible products according to explicit query criteria and documented source rules; do not silently mix incompatible releases or calibrations.
5. **Retrieve products:** download or obtain products through the verified archive access method, with timeouts, bounded retries, and size limits.
6. **Validate products:** inspect the actual file structure, readability, dimensions, units, WCS, observation time, channel/wavelength, release, flags, variance, calibration, and other required metadata.
7. **Generate cutouts/previews:** extract the requested sky region using validated WCS. Preserve the original product and record cutout parameters and processing version.
8. **Build frame manifest:** emit one manifest entry per valid observation/frame with its true observation time and artifact reference.
9. **Deliver artifacts:** stream or expose controlled preview/cutout URLs through the API.
10. **Record provenance:** persist retrieval attempts, product references, validation outcomes, checksums when computed, cutout processing, and cache origin.

A successful archive query with no matching observations is a valid empty result. It is not an archive failure, a detection, or proof that no astronomical object exists.

### 8.2 Observation epoch versus spectral channel

SPHEREx's 102 spectral channels are wavelength samples, not 102 time epochs. The backend must represent these as separate dimensions:

- **Observation epoch/time:** when a particular observation was acquired.
- **Spectral channel/wavelength:** which wavelength interval/sample the product represents.
- **Detector/array identity:** the detector/array metadata associated with the product, when available.

Do not generate an apparent time sequence by iterating through 102 channels. A playback sequence must be based on distinct, validated observation times. If the user selects a channel, the backend should return the corresponding compatible frame for each available epoch where that product exists.

### 8.3 Six detector arrays and combination behavior

If the selected SPHEREx products expose detector-array identity and NASA's suggested array combination is implemented, handle each array as documented source metadata. Detector arrays are not interchangeable with the 102 spectral channels.

Array combination/stacking is an optional scientific processing capability, not a default viewer behavior. It may run only when:

- The product documentation defines the relevant array data and combination semantics.
- Inputs are compatible in observation time, wavelength/channel, calibration, WCS, units, and required quality metadata.
- The combination method, weighting, masks, uncertainty propagation, and rejection rules are documented and validated.
- The output records all input observation/product IDs and the algorithm/configuration version.

Do not combine arrays merely because six array labels exist. If compatibility or calibration is not established, expose separate frames or report the operation as unsupported/insufficient-data.

### 8.4 Cutout generation

Cutout generation must:

- Use the product's validated WCS and correct angular units.
- Record center coordinates, coordinate frame, requested size/FOV, output pixel dimensions, and pixel scale.
- Handle RA wraparound and polar regions according to tested WCS behavior.
- Preserve masks/quality planes or explicitly link them when a preview cannot display them.
- Avoid silently resampling or changing units in a way that could affect scientific measurements.
- Record software version, parameters, warnings, and the source artifact reference.
- Distinguish an exact archive-provided cutout from a locally generated derivative.

If a product has missing or invalid WCS, the backend must not pretend that an accurate coordinate cutout was created. Return the documented failure/unsupported/insufficient-data outcome and retain the reason.

### 8.5 Preview generation and scientific source preservation

The browser may need an image format or stretch that differs from the original scientific data. Treat preview generation as a presentation transformation:

- Keep raw science data separate and immutable.
- Record stretch/normalization, scaling, channel selection, pixel transformation, output dimensions, and processing version.
- Do not apply an undocumented automatic stretch and then use preview pixel values for scientific measurements.
- Preserve masks and flags as accessible metadata or separate overlays when possible.
- Provide a clear association from every preview to its source product and cutout definition.

Do not render an invented or synthetic placeholder as if it came from SPHEREx. Fixture previews must be clearly labelled as fixtures/demo data in API metadata and UI-visible fields.

---

## 9. Image Comparison and Time-Series Services

The backend provides validated comparison results where supported. Basic timeline playback and blinking are frontend presentation functions using the backend's frame manifest.

### 9.1 Frame-to-frame compatibility

Before scientific comparison, check at least:

- Both frames are valid and readable.
- Observation timestamps are distinct and trustworthy.
- The same intended sky region is represented.
- Coordinate systems, WCS, orientation, pixel scale, and registration requirements are addressed.
- The selected channels/wavelengths and photometric quantities are compatible.
- Product releases, processing versions, calibration states, and masks/quality flags are compatible or a documented cross-release method is used.
- The relevant uncertainty, variance, PSF, and background information is available for the intended operation.

Do not interpret visible displacement as physical motion until image alignment and registration quality satisfy the criteria in `SCIENTIFIC_METHODOLOGY.md`. A visibly different image may reflect different wavelength, calibration, PSF, background, exposure, or processing rather than temporal change.

### 9.2 Image blinking

For a basic blink viewer, the backend only needs to supply valid frames and metadata. If the frames are not registered, the UI must not imply that the blinking comparison is astrometrically aligned. The backend should expose registration state/quality metadata when an alignment operation has been performed.

### 9.3 Image subtraction

Difference-image generation is allowed only when:

- Input observations are distinct, validated epochs.
- Inputs have compatible scientific meaning and selected channel/wavelength.
- They are registered to a common reference frame using a documented method.
- Relevant backgrounds, photometric scales, masks, PSF differences, and uncertainties are handled using a validated procedure.
- The output includes residual/quality information and clear limitations.

A subtraction failure must not return a visually plausible but unvalidated difference image. If the prerequisites are absent, return the contract-defined `NOT_SUPPORTED`, `INSUFFICIENT_DATA`, or blocked outcome.

### 9.4 Image addition / stacking

Image addition or stacking is not equivalent to arbitrary pixel-wise addition. It may be implemented only for scientifically compatible observations and a documented method that accounts for alignment, units, weights, exposure, masks, noise, background, and uncertainty. Preserve the full input list and combination provenance.

### 9.5 Motion and variability outputs

The backend may expose measured source positions, positional changes, flux/brightness changes, uncertainties, and significance only when the data and validated methods support them. Motion analysis requires distinct observation epochs and adequate astrometric registration. Variability analysis requires compatible photometric measurements and uncertainty handling.

Do not label a source as moving or variable solely because a user sees a difference between two images. Distinguish:

- visual difference observed in frames;
- validated measured positional/flux change;
- agent interpretation of a validated measurement;
- candidate prioritization for human review.

---

## 10. Spectral Services

The backend may support channel selection and on-the-fly spectrum generation when the selected product and measurement method permit it.

Requirements:

- Treat the 102 spectral channels as wavelength samples, not epochs.
- Read channel/wavelength values from validated product metadata rather than assuming that an integer channel index is a wavelength.
- Preserve wavelength units, flux units, calibration status, uncertainty, masks, quality flags, and source-product identifiers.
- Ensure the source aperture/region and background treatment are defined and reproducible.
- Apply only a documented extraction/combination method validated for the selected product.
- Return the spectrum with provenance, method version, input observation IDs, and missing/flagged-channel information.
- Do not interpolate missing channels without an explicit, justified, clearly labelled method.
- Do not combine different observation epochs or releases as though they were a single spectrum unless the scientific method explicitly permits it and records that choice.

The UI may request a spectrum for a selected object/region. The backend must confirm that the requested product coverage and method are supported before running the operation.

---

## 11. External Catalogue Services and Overlays

Gaia and WISE overlay support is an optional extension suggested by the NASA challenge. Do not imply that a catalogue integration exists until its source, access method, metadata, and validation are implemented.

If implemented:

- Put each catalogue behind a replaceable adapter.
- Record catalogue name, release/version, query parameters, query timestamp, coordinate reference frame/epoch, identifiers, units, and returned quality metadata where available.
- Normalize results into a documented internal schema without discarding original source identifiers.
- Apply coordinate epoch/proper-motion handling where scientifically required by the selected catalogue and query.
- Return overlay records separately from SPHEREx pixel data; do not merge catalogue positions into the underlying science image.
- Make unmatched and ambiguous outcomes explicit. No-match is not proof of novelty.
- Preserve the matching radius/method and candidate associations for audit.

Catalogue overlays are contextual visualization. Catalogue cross-matching used for a scientific candidate assessment must still follow `SCIENTIFIC_METHODOLOGY.md` and the investigation workflow's validation gates.

---

## 12. Caching, Performance, and Resource Management

The public Explorer must remain responsive without allowing unbounded archive requests or memory use.

### 12.1 Cacheable artifacts

Where allowed by source terms and operational policy, cache:

- Archive observation-discovery responses for a bounded period.
- Downloaded immutable source products, keyed by stable product identity/release/checksum when available.
- Generated cutouts and previews, keyed by source-product identity plus center, FOV, pixel size, channel, processing parameters, and processing version.
- Validated comparison products, keyed by ordered input frame IDs and full method/configuration version.

A cache key must include all parameters that affect the output. Never return a cached cutout produced for a different coordinate, FOV, channel, release, or stretch configuration.

### 12.2 Cache invalidation and source updates

- Keep product release and processing version in cache metadata.
- Do not silently serve an older release when a newer release is requested.
- Expire discovery results according to a documented TTL because archive availability can change.
- Preserve previously retrieved source products and derived artifacts when required for reproducibility; cache expiry must not delete evidence-linked artifacts.
- Make cache hit/miss and source retrieval provenance observable.

### 12.3 Bounded processing

- Apply maximum request area/FOV, frame count, channel count, download size, concurrent archive requests, and processing duration using validated configuration.
- Prefer cutouts and previews over transferring full survey products when the archive and product permit.
- Use bounded concurrency and backpressure; do not open unbounded downloads or image-processing tasks.
- Avoid loading multiple full-resolution images into memory when tiled/chunked or sequential processing is sufficient.
- Do not store large arrays or FITS payloads inside PostgreSQL rows or JSON responses.
- Support pagination or an explicit result cap for large observation lists, as finalized in `API_SPECIFICATION.md`.
- Log timing and resource usage without logging full pixel arrays or sensitive payloads.

Do not add Redis, Celery, a distributed cache, or object storage for scale alone. Start with a controlled local cache and move to another service only when real deployment requirements justify it.

---

## 13. Database and Persistence

`DATABASE_SCHEMA.md` defines exact tables, columns, constraints, relationships, and indexes. Implement only according to the canonical schema. The current persistence design includes investigations, executions, tasks, retrievals, observations, measurements, evidence, candidates, reports, and reviews. The combined Explorer also requires the schema to represent image products/artifacts and derived frames sufficiently for reliable retrieval and provenance.

### 13.1 Explorer persistence requirements

Before implementation, update `DATABASE_SCHEMA.md` to determine whether and how to persist the following logical records (do not independently create competing table names here):

- Sky-region query or query history, when persistence is useful and permitted.
- Archive observation/product references and release/processing versions.
- Observation acquisition time and time convention.
- Spectral channel/wavelength and detector/array metadata when available.
- Image artifact/cutout/preview references, storage metadata, checksum, and media type.
- WCS/coordinate metadata, image dimensions, units, and pixel-scale metadata where required.
- Quality, contamination, mask, variance, PSF, calibration, and validation references.
- Derived image product references, input frame relationships, processing method/version, and parameters.
- Cache metadata where persistent cache indexing is needed.
- Linkage between an Explorer selection and a later investigation/execution.

Do not persist the full image array in relational rows. Persist identifiers, metadata, checksums, and controlled artifact references. The actual table/column design must follow `DATABASE_SCHEMA.md`.

### 13.2 Persistence rules

- Use centralized SQLAlchemy session management.
- Keep queries in repositories/persistence services, not scattered through routes and agents.
- Use transactions for logically related writes.
- Enforce the specified foreign keys and uniqueness constraints.
- Use Alembic migrations for schema changes.
- Test migrations against temporary PostgreSQL databases.
- Keep API schemas independent from ORM structure.
- Store large FITS files/image arrays outside ordinary relational rows; persist metadata, identifiers, checksums, and file references.
- Preserve source metadata and archive IDs for auditability.
- Do not silently overwrite prior results when input releases or method versions change.
- Keep the database connection URL configurable and document backup/restore.
- Separate durable evidence-linked artifacts from disposable cache artifacts.

PostgreSQL is the canonical relational database for local development, CI, staging, and production. No alternate relational database is configured as an application fallback. Keep transactions short, enforce the foreign-key and integrity constraints defined in `DATABASE_SCHEMA.md`, and do not assume unlimited concurrent writes or multi-instance coordination. Configure connection pooling and database timeouts through deployment configuration, coordinated with the application worker count.

---

## 14. Canonical Statuses and Error Semantics

Use the status values in `API_SPECIFICATION.md`, `DATABASE_SCHEMA.md`, and `INVESTIGATION_WORKFLOW.md`. Project baseline:

**Task statuses:** `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRYING`, `CANCELLED`, `BLOCKED`, `NOT_SUPPORTED`, `INSUFFICIENT_DATA`.

**Quality outcomes:** `PASSED`, `FAILED`, `INCONCLUSIVE`, `NOT_AVAILABLE`, `NOT_APPLICABLE`.

**Evidence gates:** `PASSED`, `FAILED`, `UNKNOWN`, `NOT_APPLICABLE`.

**Candidate statuses:** `NEEDS_MORE_DATA`, `NOT_PRIORITIZED`, `PRIORITIZED_FOR_REVIEW`, `BLOCKED_BY_QUALITY_ISSUE`.

Use exact source-document enums; do not invent synonyms like `DONE` or `SUCCESS`.

### 14.1 Explorer retrieval outcomes

The API contract must distinguish at least:

- Valid query with matching observations.
- Valid query with zero matching observations.
- Unsupported query/product/channel/release.
- Insufficient coverage or metadata for the requested operation.
- Archive timeout/provider error.
- Product missing, malformed, unreadable, or corrupt.
- Cutout/preview generation failure.
- Quality failure or inconclusive quality result.
- Artifact not found or no longer available.
- Request rejected by configured resource limits.

Do not invent a new public enum in this document. Map these conditions to the canonical error/status model in `API_SPECIFICATION.md`.

### 14.2 Task transition rules

- New tasks begin `QUEUED`.
- `RUNNING` means the orchestrator has begun execution.
- `SUCCEEDED` requires completed work and valid persisted output.
- `FAILED` records a failure category and safe diagnostic.
- `RETRYING` is limited to retryable failures and bounded attempts.
- `CANCELLED` is never success.
- `BLOCKED` means a prerequisite, policy, or gate prevents work.
- `NOT_SUPPORTED` means the requested capability is not supported.
- `INSUFFICIENT_DATA` means the available inputs cannot justify a result.
- A successful zero-result query is distinct from a failed query.
- Natural-language agent output alone cannot make a task successful.
- Preserve the reason and provenance of terminal outcomes.

---

## 15. Workflow Orchestrator

The Workflow Orchestrator is a deterministic system component, **not an agent**. It owns task dependencies, order, state transitions, bounded retries, cancellation, and evidence-gate enforcement for investigations.

Responsibilities:

1. Accept a validated investigation request.
2. Create investigation and execution records.
3. Resolve requested analysis against a capability registry.
4. Build a bounded plan using the approved workflow definition.
5. Create tasks with IDs, dependencies, input and output contracts.
6. Dispatch tasks in valid order.
7. Record starts, outputs, failures, retries, and completion times.
8. Block downstream analysis when required prerequisites fail.
9. Apply timeouts and safe retry policies.
10. Handle cancellation and repeated requests according to the API contract.
11. Persist evidence/provenance references.
12. Trigger report assembly only from available persisted results.
13. Preserve partial failures instead of falsely reporting total success.
14. Expose progress for the frontend.

Start with one explicit, single-process workflow. Do not add Kafka, Celery, Redis, or microservices without a documented need. Distinguish total success, critical failure, partial results, blocked tasks, unsupported work, insufficient data, and successful zero-result queries.

### 15.1 Explorer is not an investigation execution

Observation discovery and basic frame retrieval are Explorer services. They must not automatically create agent runs or candidate records. A user action must explicitly request an investigation.

If the user starts an investigation from the Explorer, create a new investigation/execution and snapshot the selected input references and parameters. The investigation must be reproducible even if the user later changes the viewer's selected frame or FOV.

The orchestrator must revalidate that the selected observations still exist, belong to compatible products/releases, and satisfy the requested method's prerequisites. A client-provided frame ID or query result is a reference, not proof that the input is valid.

### 15.2 Capability registry

Keep a backend capability registry describing what is currently supported. It should distinguish at least:

- Observation discovery.
- Image product retrieval.
- WCS-aware cutout generation.
- Preview generation.
- Frame-manifest generation.
- Image blinking support (frame availability; playback remains frontend behavior).
- Image registration.
- Difference-image generation.
- Image stacking/addition.
- Source detection and tracking.
- Motion measurement.
- Photometry/variability analysis.
- Spectral extraction/comparison.
- Catalogue overlay/cross-match.

A capability should be advertised as enabled only when its data requirements, implementation, and relevant tests are in place. Keep `planned`, `unavailable`, or `not validated` distinct from `supported`. Capability discovery responses must follow `API_SPECIFICATION.md`.

---

## 16. Agent Contracts

The eight logical agent roles are:

1. Investigation Planner Agent.
2. Data Retrieval Agent.
3. Data Quality Agent.
4. Change Detection Agent.
5. Motion Analysis Agent.
6. Spectral & Photometry Agent.
7. Catalogue Matching Agent.
8. Validation & Hypothesis Agent.

Their detailed responsibilities must follow `AGENT_SPECIFICATIONS.md`. Each agent receives typed inputs and allowed evidence references and returns structured output validated by code.

Every invocation should identify `investigation_id`, `execution_id`, `task_id`, agent name/version, input schema, allowed tools, output schema, and timeout/resource limits as appropriate.

Agents may plan within supported capabilities, request approved retrievals, interpret validated quality summaries, propose documented methods, explain computed outputs, compare hypotheses using supplied evidence, and draft a readable summary.

Agents must never:

- Invent measurements, uncertainties, source IDs, timestamps, units, archive results, image pixels, or catalogue matches.
- Run arbitrary shell commands, SQL, Python, or filesystem operations based on generated text.
- Bypass gates or change persisted measurements.
- Treat catalogue no-match as proof of novelty.
- Treat spectral channels as temporal observations.
- Claim Planet X is confirmed.
- Turn failed retrieval into a successful empty query.
- Hide failed/inconclusive gates.
- Change candidate status outside the Candidate Assessment Service.

| Operation | Owner |
|---|---|
| Explorer query validation and limits | Explorer service + API validators |
| Archive access | Data-source adapter |
| Image artifact creation/delivery | Image artifact service |
| Metadata/quality validation | Deterministic validators |
| Image registration/comparison | Scientific Engine |
| Measurements and statistics | Scientific Engine |
| Workflow order and task states | Workflow Orchestrator |
| Candidate status/ranking | Candidate Assessment Service |
| Human-readable interpretation | Agent/reporting layer using validated records |
| Final scientific confirmation | Human researchers |

The agents do not need to run for every Explorer request. The Data Retrieval Agent may assist an investigation plan, but ordinary coordinate search and image viewing must be handled by deterministic services and adapters.

LLM failure must not corrupt results. If summary generation fails, return the available scientific records and explicitly identify the summary failure.

---

## 17. Scientific Engine and Evidence Gates

Scientific calculations and image-derived measurements must be deterministic, versioned, tested Python code—not LLM output.

For each implemented method, document purpose, inputs, preconditions, units, time/coordinate conventions, algorithm version, output schema, uncertainty treatment, quality conditions, failure behavior, limitations, and a known-answer test with justified tolerance.

### 17.1 Non-negotiable safeguards

- Never replace missing values with plausible measurements.
- Validate finite numbers, units, coordinate frames/WCS, and timestamps as required.
- Temporal analysis requires distinct validated acquisition times.
- Wavelength/channel number is not time.
- Do not compare incompatible data releases/calibration states without a validated method.
- Respect quality flags, variance, masks, PSF, and calibration metadata.
- Keep uncertainty and quality outcomes with measurements.
- Return `NOT_SUPPORTED` or `INSUFFICIENT_DATA` when preconditions fail.
- Preserve references needed to reproduce outputs.
- Label synthetic/demo results clearly.
- Do not infer physical motion from unregistered or incompatible images.
- Do not present a display preview as the authoritative measurement input.

### 17.2 Six investigation gates

The project identifies six gates; implement exact thresholds only from `SCIENTIFIC_METHODOLOGY.md` and `INVESTIGATION_WORKFLOW.md`:

- **G1 — Motion**
- **G2 — Persistence**
- **G3 — Flux/Spectrum**
- **G4 — Cross-match**
- **G5 — Quality**
- **G6 — Evidence readiness**

Each gate returns an explicit outcome, supporting evidence references, and a reason. `UNKNOWN` is not a pass. A gate irrelevant to the bounded MVP should be `NOT_APPLICABLE` with a reason, not falsely marked as passed.

### 17.3 Evidence

An evidence record should link, according to the canonical schema, the input product/observation, retrieval, task/execution, method and version, measured values and units, uncertainty, quality/gate result, warnings, limitations, preview or derived-artifact references where relevant, and provenance.

### 17.4 Candidate assessment

The Candidate Assessment Service is a deterministic system service, not an agent. It applies documented, versioned criteria to validated evidence and gate outcomes. Candidate states:

- `NEEDS_MORE_DATA`
- `NOT_PRIORITIZED`
- `PRIORITIZED_FOR_REVIEW`
- `BLOCKED_BY_QUALITY_ISSUE`

`PRIORITIZED_FOR_REVIEW` means the candidate merits human attention; it is not a discovery or confirmation.

---

## 18. SPHEREx / IRSA Adapter

`DATA_SOURCE_SPEC.md` is authoritative. The primary mission is SPHEREx and the primary archive is NASA/IPAC IRSA.

The data-source specification reviewed on 2026-10-21 says to prefer QR3 where available and appropriate, use QR2 only when explicitly required or needed data are unavailable in QR3, never silently mix QR2 and QR3 measurements, and not treat retired QR1 as current. Re-check current archive documentation during implementation because release status, products, and interfaces can change.

### 18.1 Adapter responsibilities

1. Accept a validated query.
2. Use a documented archive access route verified against current official product documentation.
3. Record exact query parameters and retrieval time.
4. Distinguish success, valid zero results, unavailable product, timeout, malformed response, and provider failure.
5. Normalize only documented metadata.
6. Preserve product ID/filename, release, processing version, observation ID/time, units, WCS, channel/wavelength, detector/array metadata where available, quality flags, variance/PSF references, calibration information, checksum, and documentation reference where available.
7. Validate downloaded files' actual structure and size.
8. Store large data files outside relational rows and persist references/metadata.
9. Return structured retrieval records with warnings/errors.
10. Never invent missing metadata or interpret no-match as a discovery.

Before analysis or cutout creation, validate file readability, expected structure, dimensions, units, WCS, times, wavelength metadata, detector metadata where relevant, quality/variance interpretation, release, and processing version as applicable. Preserve unfamiliar source metadata or flag it for review; do not silently discard it.

### 18.2 Adapter interface

Illustrative interface only; adapt to the actual typed contracts in `DATA_SOURCE_SPEC.md` and `API_SPECIFICATION.md`:

```python
class DataSourceAdapter(Protocol):
    def discover(self, query: DataQuery) -> RetrievalResult: ...
    def retrieve(self, product: ProductReference) -> RetrievedProduct: ...
    def validate(self, product: RetrievedProduct) -> ProductValidationResult: ...
    def create_cutout(
        self,
        product: RetrievedProduct,
        region: SkyRegion,
    ) -> CutoutResult: ...
```

The fixture adapter must implement the same normalized contract so tests can run without network access. Use sync or async I/O consistently with the actual implementation.

### 18.3 Live-data verification

Before claiming live SPHEREx support:

- Verify the exact current archive access route and product documentation.
- Verify that the selected target/region has the required product coverage.
- Retrieve a real product and validate its FITS structure and metadata.
- Confirm actual observation timestamps and whether distinct epochs are available.
- Confirm that cutouts can be created using the real WCS.
- Record product release and all relevant quality/calibration metadata.
- Test zero-results, timeout, provider failure, unavailable products, and malformed responses.

A fixture test passing does not prove live archive integration. A live retrieval for one coordinate does not prove coverage for all coordinates or all five NASA target objects.

---

## 19. NASA Challenge Target Validation

The backend acceptance suite must include the five objects named by the NASA challenge. Target definitions and coordinates must be sourced from the NASA challenge brief or verified astronomical references and then stored in a versioned test configuration. Do not invent coordinates for targets whose coordinates are not supplied in the brief.

Required target set:

1. Barnard's Star.
2. Epsilon Indi B (the brown-dwarf pair associated with the system).
3. WISE 0855 — challenge-supplied coordinate: RA `08h55m10.831s`, Dec `-7d14m42.53s`.
4. RW Aurigae.
5. R Draconis.

For each target, tests should verify as applicable:

- Target-name/coordinate resolution is valid and provenance-tracked, if name resolution is implemented.
- The sky-region query validates and returns a truthful availability result.
- The archive query can distinguish observations found, valid zero results, and provider failure.
- Any returned frames have actual observation times, valid product IDs, WCS, release, channel/wavelength metadata, and quality flags where available.
- The image manifest is ordered by actual acquisition time.
- The viewer can retrieve the corresponding cutouts/previews.
- Where the target is expected to exhibit motion or variability, the scientific test uses a documented method and known/reference expectations; do not assert a positive signal without real validated data.

**Critical:** NASA's instruction to test the tool on these targets does not authorize fabricating observation epochs or guaranteeing that every target has multiple usable SPHEREx observations in the current archive release. If the real archive does not provide the needed coverage, record the limitation and test the correct zero-result/insufficient-data behavior. Do not silently substitute non-SPHEREx images while labelling them as SPHEREx. External catalogue context may be shown separately with clear source labels.

---

## 20. Reports

The Scientific Report Builder is a system service, not an agent. Assemble reports from persisted validated records. An LLM may improve readability but cannot add facts absent from evidence.

Include, where supported:

- Investigation objective and scope.
- Selected sky region, coordinate frame, FOV, and observation-time range.
- Data source, product release, channel/wavelength, and retrieval details.
- Exact observation frames and source product references used.
- Methods and software/method versions.
- Measurements with units and uncertainty.
- Image registration/comparison details and quality outcomes, where applicable.
- Six-gate results and evidence references.
- Candidate criteria and rationale.
- Missing inputs, failures, unsupported operations, and warnings.
- Alternative hypotheses and unresolved questions.
- Reproducibility references.
- Explicit synthetic/demo versus archive-derived label.
- Human-review state and limitations.

Separate observed facts, computed measurements, interpretation, and hypotheses. Never claim a planet is confirmed based solely on a catalogue no-match or candidate score. A report launched from the Explorer must preserve the exact input frame references and query parameters used for the investigation.

---

## 21. Errors, Retries, and Idempotency

Use the exact public error envelope and codes in `API_SPECIFICATION.md`. Internally distinguish invalid input, unsupported capability, insufficient data, successful zero results, provider failure/timeout, corrupt product, invalid/missing WCS, cutout failure, quality failure/inconclusive result, scientific computation failure, persistence failure, LLM failure, cancellation, artifact-not-found/expired, rate-limit/resource-limit rejection, and unexpected internal errors.

- Retry only transient failures and only within a bounded policy.
- Do not retry invalid input or deterministic scientific validation failures without changed inputs.
- Record each attempt.
- Respect API idempotency behavior; do not create duplicate executions for repeated requests when the contract promises idempotency.
- Do not blindly retry non-idempotent actions.
- Preserve earlier failure provenance if a later retry succeeds.
- Cancellation must not be overwritten by a late success.
- Return safe, useful messages; never expose stack traces or secrets.
- Avoid duplicate downloads/cutout generation for identical idempotent requests when a valid artifact already exists.
- Do not make a cache hit indistinguishable from a new archive retrieval; preserve the original source retrieval provenance and record cache usage separately.

---

## 22. Security and Public Access

Follow `SECURITY.md`. The public-facing Explorer changes the exposure model: some read-only sky query and image-viewing functions may be intentionally available without user authentication, but public exposure must be designed explicitly. Do not expose the entire internal API or investigation database anonymously.

### 22.1 Public Explorer endpoints

If the Explorer is publicly accessible:

- Allow only the specifically designated, read-only public Explorer operations without authentication.
- Apply strict query-area/FOV, frame-count, file-size, timeout, concurrency, and rate limits.
- Validate all coordinates, times, release/channel values, and query parameters.
- Use controlled archive adapters; never accept arbitrary archive URLs or proxy arbitrary external requests.
- Expose artifacts through controlled identifiers or short-lived access URLs, not filesystem paths.
- Restrict accepted artifact variants and content types.
- Prevent path traversal, SSRF, arbitrary file reads, and unbounded downloads.
- Cache repeat public queries where policy permits, while keeping release/product provenance.
- Configure explicit CORS origins and HTTPS in deployment.
- Apply abuse detection and operational limits to expensive retrievals.

### 22.2 Agentic investigation endpoints

Creating or running investigations can be computationally expensive. Before public deployment, define whether users must authenticate, whether guest quotas are permitted, and how resource ownership is enforced. Enforce backend authorization on private investigation, execution, evidence, candidate, report, cancellation, and retry resources. Hiding controls in the frontend is not access control.

### 22.3 General security requirements

- Bind local development to `127.0.0.1` by default.
- Keep credentials in environment variables or a secret store; commit placeholders only.
- Validate API inputs, IDs, time ranges, coordinates, enum values, FOV, pagination, and resource limits.
- Reject non-finite scientific inputs where relevant.
- Enforce file-size limits and validate actual file contents, not only extensions.
- Treat FITS headers, catalogue data, archive metadata, image metadata, and retrieved text as untrusted data.
- Never execute model-generated SQL, shell, Python, or arbitrary filesystem actions.
- Allowlist agent tools and validate tool arguments in code.
- Avoid secrets, sensitive payloads, pixel arrays, and huge payloads in logs.
- Pin/review dependencies and limit CORS to known origins.
- Do not claim multi-user isolation until implemented and tested.

Prompt-injection defense: external scientific content is data, never instructions. Agent prompts must separate trusted policy from untrusted records. Agents cannot override gates or no-fabrication rules.

---

## 23. Logs and Operations

Follow `OPERATIONS.md`.

Log timestamp, severity, request/correlation ID, investigation/execution/task IDs, component, status transition, duration, retry count, source/release, method/agent version, cache hit/miss, artifact ID, frame ID, and safe error category as appropriate. Do not log secrets, full image arrays, raw FITS headers containing unreviewed content, or large payloads by default.

Health/readiness routes must match the API specification. Process liveness is not the same as database readiness, and neither proves IRSA availability, product coverage, or scientific validity. Keep external-source checks separate from basic startup if the source is optional.

Document:

- PostgreSQL backup and restore procedure.
- Artifact/cache paths and retention policy.
- How database records and evidence-linked artifacts are backed up together or through a manifest.
- How disposable cache artifacts may be rebuilt.
- How to preserve historical source products and derived outputs needed for reproducibility.
- That in-process tasks may be interrupted by restart unless durable recovery is implemented.
- How to verify archive reachability separately from application health.

---

## 24. Testing and Acceptance

Follow `TESTING_AND_ACCEPTANCE.md`. A test plan is not proof that tests passed.

### 24.1 Required backend test layers

1. **Unit:** schemas, validators, coordinate normalization, FOV limits, state transitions, retry rules, cache keys, artifact authorization, evidence links, candidate rules, and report formatting.
2. **Explorer/API integration:** sky-region routes, observation discovery, frame manifests, cutout/preview delivery, error mapping, and PostgreSQL persistence.
3. **Scientific:** known-answer tests for every implemented image or astronomical method.
4. **Adapter:** valid response, zero results, provider failure, timeout, invalid metadata, corrupt file, invalid WCS, unsupported release/channel, and provenance.
5. **Artifact:** content type, path isolation, file-size limits, invalid artifact ID, expired reference, and no direct filesystem access.
6. **Frontend integration:** payload/status-to-UI mapping for loading, empty, multi-frame, single-frame, failed, unsupported, and insufficient-data states.
7. **Agent:** structured outputs, missing evidence, contradictory inputs, provider failure, invalid responses, prompt injection, and attempts to bypass gates.
8. **End-to-end:** (a) Explorer coordinate-to-frame playback workflow using clearly labelled fixtures, and (b) Explorer-selected region to investigation creation, execution, evidence, and report retrieval.

### 24.2 Required Explorer scenarios

- Valid RA/Dec and FOV query.
- Invalid RA/Dec, non-finite values, invalid coordinate frame, and out-of-range FOV.
- RA wraparound and relevant polar-edge cases.
- Valid query with zero observations.
- Multiple observations at distinct actual times.
- Only one observation available; viewer must not imply temporal change.
- Duplicate observation records and duplicate retrieval request.
- Missing or invalid WCS.
- Corrupt/unreadable FITS or image product.
- Unsupported channel/release.
- Mismatched product release or calibration state.
- Missing uncertainty/variance/quality metadata.
- Cutout/preview generation and artifact retrieval.
- Artifact access attempt with an invalid or unauthorized identifier.
- Large query/frame request blocked by configured limits.
- Archive timeout, provider error, and retry behavior.
- Cache hit, cache miss, expiry, and invalidation when release/processing version changes.
- Image sequence ordering by observation time rather than channel number.
- Difference/addition blocked when compatibility or registration prerequisites fail.
- Preview transformation cannot be mistaken for the authoritative science image.
- Shareable coordinate URL round-trip at the API/query-contract boundary.

### 24.3 NASA challenge target scenarios

Create explicit tests for Barnard's Star, Epsilon Indi B, WISE 0855 (using the challenge-provided coordinates), RW Aurigae, and R Draconis. Verify actual archive availability and the correct behavior for each result. Do not require a fabricated positive detection or assume every target has multiple usable epochs. For target-name resolution, record the authoritative source of coordinates and proper-motion/epoch treatment where relevant.

### 24.4 Agentic investigation scenarios

Retain the existing investigation tests:

- Investigation creation and retrieval.
- Execution/task status and dependency handling.
- At least one supported deterministic scientific analysis.
- Evidence/provenance retrieval.
- Candidate assessment with known match and no-match cases.
- Report generation from persisted evidence.
- Missing metadata, corrupt inputs, unsupported analysis, insufficient data, quality failure, cancellation, retry, duplicate execution, provider failure, malformed agent output, and contradictory evidence.

### 24.5 Release blockers

- Clean setup starts the backend.
- Alembic migrations work on a clean PostgreSQL database.
- API/OpenAPI matches the updated API contract.
- Invalid Explorer and investigation requests fail safely.
- Public endpoint limits and artifact access controls are enforced.
- A fixture-backed coordinate-to-observation-to-image-frame flow works end to end.
- The frontend can retrieve and navigate the returned frame sequence using actual frame metadata.
- Live IRSA/SPHEREx support is claimed only after a real product retrieval, metadata validation, and cutout test pass.
- The five NASA challenge targets have explicit availability/validation test records; missing coverage is reported honestly.
- Every implemented scientific method has a deterministic test.
- Evidence traces to inputs and method versions.
- Unsupported and insufficient-data cases are explicit.
- No-match is not treated as discovery.
- Explorer selections can launch a bounded agentic investigation without trusting client-supplied measurements.
- Frontend can retrieve progress, evidence, and report states.
- Secrets are neither committed nor leaked.
- Documentation distinguishes implemented, fixture-only, live-verified, planned, and unavailable features.

---

## 25. Backend Implementation Order

This section defines backend work sequence. The overall project sequence remains governed by `IMPLEMENTATION_PLAN.md`.

### Step 0 — Contract and scientific feasibility

Review `PRD.md`, `SYSTEM_ARCHITECTURE.md`, `DATA_SOURCE_SPEC.md`, `SCIENTIFIC_METHODOLOGY.md`, `API_SPECIFICATION.md`, `DATABASE_SCHEMA.md`, `UI_UX_SPECIFICATION.md`, `SECURITY.md`, and `TESTING_AND_ACCEPTANCE.md`. Resolve the combined Explorer + Agentic product contracts. Verify the exact SPHEREx product release, archive access route, fields, units, timestamps, WCS, flags, and quality metadata. Select fixtures and expected outputs. Record limitations.

**Exit:** the Explorer data path and at least one bounded investigation path are specified; all claims are supported by verified data or clearly labelled fixtures.

### Step 1 — Backend foundation

Set the canonical package layout, project configuration, dependencies, `.env.example`, `.gitignore`, settings, logging, FastAPI entrypoint, health/readiness, and pytest.

**Exit:** clean setup starts the API; secrets and runtime artifacts are excluded from Git.

### Step 2 — Shared contracts and persistence

Update typed contracts and the canonical database schema for observation/product references, image artifacts, frame metadata, and Explorer-to-investigation linkage. Implement PostgreSQL sessions, models, Alembic migrations, repositories, and migration tests.

**Exit:** typed observation and artifact metadata can be persisted and retrieved without putting image arrays in relational rows.

### Step 3 — Fixture data-source adapter

Implement the normalized adapter contract, fixture product metadata, observation discovery, retrieval provenance, zero-result/error distinctions, and invalid/corrupt fixture tests.

**Exit:** the adapter works without network access and never fabricates live mission data.

### Step 4 — Explorer query and frame-manifest services

Implement sky-coordinate/FOV validation, observation discovery service, frame manifest generation, time ordering, and API integration according to the finalized API contract.

**Exit:** a valid fixture query returns a correct frame manifest, including a truthful zero/single/multiple-frame result.

### Step 5 — Image artifact and cutout delivery

Implement controlled artifact storage, WCS-aware cutout generation, preview generation, provenance, caching, size limits, and controlled image delivery.

**Exit:** the frontend can retrieve a valid fixture-derived cutout/preview without direct filesystem access.

### Step 6 — Deterministic image/scientific engine

Implement only justified methods such as registration, image difference, source detection, motion, photometry, or spectral extraction. Define preconditions, units, uncertainty, quality outputs, and known-answer tests for each included method.

**Exit:** each included method passes deterministic validation; unsupported operations are rejected explicitly.

### Step 7 — Investigation, evidence, and candidate assessment

Implement the bounded orchestrator, evidence/provenance persistence, versioned candidate rules, retries/cancellation, and report assembly.

**Exit:** an Explorer-selected region/observation set can start a reproducible investigation and produce a report traceable to the selected inputs.

### Step 8 — Agent integration

Add the eight logical agents only where they provide value. Enforce typed contracts, tool allowlists, timeouts, and evidence references; isolate provider-dependent code.

**Exit:** agent failure cannot fabricate, overwrite, or corrupt scientific results.

### Step 9 — Live IRSA/SPHEREx integration

Implement a verified documented access route, release-aware metadata checks, timeouts, bounded retries, retrieval provenance, real FITS validation, and WCS-aware cutout generation. Verify actual observation-time coverage for the five NASA challenge targets.

**Exit:** the adapter passes `DATA_SOURCE_SPEC.md` and Explorer acceptance criteria. Do not claim live support before this.

### Step 10 — Optional enhanced services

Add compatible image addition/subtraction/stacking, detector-array combination, on-the-fly spectrum generation, and Gaia/WISE overlays only when their data requirements, product semantics, methods, API contracts, and tests are defined.

**Exit:** each optional capability is either validated and advertised as supported or clearly reported as unavailable/not supported.

### Step 11 — Release review

Run tests and migration checks; inspect secrets, public endpoint limits, artifact security, logging, cancellation, cache invalidation, recovery, and document consistency.

**Exit:** all applicable Explorer and investigation acceptance blockers pass.

---

## 26. Definition of Done

A backend feature is complete only when:

- [ ] It has an owner module and typed interface.
- [ ] It matches the relevant source-of-truth documents.
- [ ] Request/response schemas and errors are explicit.
- [ ] State transitions and capability checks are validated.
- [ ] Persistence changes have migrations.
- [ ] Unit and integration tests cover success and failure.
- [ ] Scientific methods have known-answer tests.
- [ ] Provenance, units, uncertainty, quality outcomes, and source references are preserved where applicable.
- [ ] Unsupported/insufficient-data outcomes are explicit.
- [ ] Artifact delivery does not expose filesystem paths or bypass limits.
- [ ] Logs do not leak secrets, raw image arrays, or huge payloads.
- [ ] API-visible changes are reflected in the frontend contract.
- [ ] Setup/operations docs are updated.
- [ ] Features are called implemented only after working and tested.

### 26.1 Explorer completion

The Explorer backend is complete for a defined release only when a user can submit a sky-region query; receive a truthful list of available SPHEREx observations; retrieve valid image frames/cutouts with actual observation times and source metadata; navigate a frame manifest; inspect quality/provenance; and receive explicit behavior for zero results, single-frame coverage, unsupported options, retrieval failures, and corrupt products.

Basic play/pause and timeline interactions are frontend behavior, but the backend must supply correct image frames and metadata. No positive coverage or scientific discovery is implied by the viewer being functional.

### 26.2 Agentic investigation completion

The investigation backend is complete when one bounded supported investigation can be created through the API, executed through the deterministic orchestrator, backed by validated source or fixture data, persisted in PostgreSQL, inspected through task/evidence records, and returned as a traceable report—with limitations represented honestly.

### 26.3 Integrated product completion

The combined backend is complete when a user can explore a supported SPHEREx region without launching an investigation, then explicitly start an investigation from selected Explorer context and trace every resulting measurement and report claim back to the exact source observations, methods, and versions.

---

## 27. Non-negotiable Rules

1. No fabricated science or image data.
2. No unsupported implementation claims.
3. Catalogue no-match is not discovery.
4. Spectral channels are not time epochs.
5. Observation timestamps must be real, validated acquisition times.
6. Unknown/failed evidence gates never silently pass.
7. Results trace to inputs, releases, methods, versions, and evidence.
8. Candidate prioritization is not scientific confirmation.
9. Frontend, API, database, orchestrator, Explorer, and report use consistent contracts.
10. Build a reliable coordinate-to-image Explorer path and a reliable bounded investigation path before scaling infrastructure or agent count.
11. A public image viewer does not require an agentic run.
12. An agentic investigation launched from the Explorer must revalidate all input references and never trust client-supplied measurements.
13. Return `NOT_SUPPORTED` or `INSUFFICIENT_DATA` rather than unjustified results.
14. Never conflate spectral channel, detector array, observation epoch, retrieval time, and product release.
15. Never imply that every NASA challenge target has available multi-epoch SPHEREx coverage until verified from the archive.

---

**End of document.**
