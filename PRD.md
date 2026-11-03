
# HELIOS-X — Product Requirements Document (PRD)

**Document:** `PRD.md`  
**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Version:** 2.0 — NASA SPHEREx Challenge Alignment  
**Status:** Proposed product requirements baseline  
**Product type:** Public-facing astronomical image explorer with an integrated agentic scientific investigation platform  
**Primary mission:** NASA SPHEREx  
**Primary archive:** NASA/IPAC Infrared Science Archive (IRSA)  
**Primary client:** React, TypeScript, Vite  
**Backend:** Python, FastAPI, Pydantic  
**Persistence:** PostgreSQL, SQLAlchemy 2.x, Alembic  

> This document defines the intended product behavior. It does not claim that the application, archive access, scientific methods, agents, or acceptance tests have already been implemented or validated.

---

## 1. Executive Summary

HELIOS-X is a public-facing SPHEREx sky exploration and scientific investigation platform. It combines two connected product capabilities:

1. **SPHEREx Sky Image Explorer — the primary public experience and NASA challenge deliverable.** Users enter a sky position using Right Ascension (RA) and Declination (Dec), define a field of view (FOV), and inspect available SPHEREx images of that region from different actual observation times. The images are presented as a time-ordered sequence with playback and frame-navigation controls so that users can visually identify possible changes in position or brightness.
2. **Agentic Scientific Investigation Engine — the integrated analysis layer.** Users can explicitly promote a selected sky region, observation sequence, or potential source into a bounded investigation. The system validates the data, runs supported deterministic scientific analyses, coordinates specialized AI-agent roles, evaluates alternative explanations, preserves evidence and provenance, prioritizes candidates for human review, and generates traceable reports.

The two capabilities share data-source adapters, scientific processing, metadata validation, artifact storage, evidence, and provenance services. They remain separate user journeys: a person must be able to explore images without creating an investigation or invoking an AI agent. Agentic analysis is an additional capability, not a prerequisite for image browsing or playback.

SPHEREx provides 102 near-infrared spectral channels. These are spectral samples, not 102 observation epochs. Time-series playback must be based on actual, distinct acquisition timestamps returned by the archive. The mission's broad six-month all-sky survey cadence must not be interpreted as a guarantee that every requested sky position has exactly one usable observation every six months. Actual regional coverage, product availability, quality, and cadence must be discovered and validated from the data.

HELIOS-X does not automatically establish astronomical discoveries. It helps people inspect observations and helps researchers investigate measurable changes. All quantitative measurements must come from deterministic, tested computational functions. AI agents may plan and interpret validated results, but may not invent observations, measurements, uncertainties, timestamps, catalogue matches, or scientific conclusions.

**Product positioning:** *Explore the changing sky. Investigate what changed. Trace every conclusion to evidence.*

## 2. Challenge Context and Problem Statement

### 2.1 NASA SPHEREx challenge context

The NASA challenge asks for a public-facing web tool that displays SPHEREx sky images and makes it easy for anyone to see how a selected patch of sky changes over time. The user supplies RA and Dec; the tool retrieves the available images for that region and presents them as a sequence resembling a small video.

The challenge describes SPHEREx as mapping the entire sky on an approximately six-month cycle in 102 bands of near-infrared light and returning imagery covering more than a billion astronomical objects. The data may contain stars, brown dwarfs, asteroids, comets, quasars, supernovae, galaxies, and other sources. The volume and complexity make direct manual inspection difficult.

The challenge specifically calls for testing the tool on:

- Barnard's Star.
- Epsilon Indi B, a brown-dwarf pair associated with a star.
- WISE 0855, with challenge-supplied coordinates RA `08h55m10.831s`, Dec `-7d14m42.53s`.
- RW Aurigae, a variable star.
- R Draconis, a variable star.

These are validation targets, not guaranteed detections. The application must test their coordinate resolution and actual archive coverage and must honestly represent zero, one, or multiple usable observations. It must not fabricate frames or assume that the data necessarily resolve every component of a multiple system.

### 2.2 Problem

Astronomical observations are distributed across large archives, products, releases, channels, observation times, and scientific tools. Users need a simple way to retrieve and visually compare images of the same sky region. Researchers additionally need a disciplined way to determine whether an apparent change is real or can be explained by noise, calibration, image-registration error, contamination, blending, source confusion, or other artifacts.

A conventional image gallery is insufficient because it may not preserve actual observation times, coordinate alignment, data-release information, quality flags, or scientific provenance. A purely agentic investigation dashboard is also insufficient for the NASA challenge because it does not, by itself, provide the requested public image-sequence experience.

HELIOS-X addresses both needs through a simple public Explorer and a deeper evidence-driven investigation workflow built on the same validated data foundation.

### 2.3 Core user problem

A user should be able to answer:

- What SPHEREx observations are available for this sky position or region?
- What did this region look like at each actual observation time?
- Can I play, pause, step through, or blink between the available images?
- Is a visible change likely to be positional, photometric, spectral, instrumental, or uncertain?
- What metadata, quality flags, and data limitations apply to each image?
- Can I start a deeper investigation and trace its results to the exact observations and methods used?

## 3. Product Vision and Product Model

### 3.1 Two integrated capabilities

| Capability | Primary purpose | User outcome | Dependency on the other capability |
|---|---|---|---|
| **Sky Image Explorer** | Search and visually inspect available SPHEREx observations for a sky region | Understand how the same region appears across actual observation times | Must work independently; does not require agents or an investigation record |
| **Agentic Investigation Engine** | Analyze a selected region/source using validated methods and controlled agent roles | Receive an evidence-linked scientific assessment and report | May start from Explorer context, but must also support other documented investigation inputs where appropriate |

### 3.2 Product hierarchy

The default entry point is the **Sky Image Explorer**. The user can then choose to investigate a region or source. The application should use progressive disclosure: basic image exploration remains simple, while technical measurements, agent activity, candidate assessments, and evidence details are available when the user chooses the investigation workflow.

The product must not force every visitor through a complex investigation form before showing images. It must not invoke all eight agent roles simply to retrieve or play images.

### 3.3 Product surfaces

1. **Public Explorer:** coordinate search, FOV selection, observation discovery, image-sequence viewer, basic metadata, and clear data-availability states.
2. **Investigation workspace:** investigation creation, run monitoring, scientific evidence, candidate assessment, and report review.
3. **Capabilities and information area:** supported data sources, products, channels, current release support, limitations, and help explaining how to use the Explorer.

The public Explorer should be usable without requiring an account. Saved/private investigations, reviewer actions, and other non-public resources must follow the authentication and authorization policy in `SECURITY.md`. A challenge demonstration may provide clearly labelled example investigations for visitors who do not have an account.

## 4. Product Goals

### 4.1 NASA challenge goals

1. Provide a public-facing web interface for exploring SPHEREx sky imagery.
2. Let users specify a sky position using RA and Dec.
3. Let users define the size of the requested sky region through a supported FOV control.
4. Discover and display the actual available observations for the requested region.
5. Present compatible observations in acquisition-time order as an interactive image sequence.
6. Make it easy to play, pause, step through, and visually compare the available frames.
7. Display meaningful observation and product metadata, including quality or contamination information where supplied.
8. Handle regions with zero, one, or multiple usable observations honestly.
9. Validate the Explorer using all five NASA-specified targets.
10. Make the data-source integration maintainable as new documented SPHEREx releases become available.

### 4.2 Scientific investigation goals

11. Allow users to explicitly start an investigation from an Explorer region, selected observation sequence, or source.
12. Verify that the selected data support each requested analysis before execution.
13. Provide deterministic, reproducible image and scientific analysis where the data and methods justify it.
14. Use the eight specialized AI-agent roles for planning, coordination, and interpretation where useful, without requiring eight separate LLM processes.
15. Evaluate relevant data-quality issues and alternative explanations.
16. Preserve evidence, uncertainty, provenance, method versions, and historical analysis results.
17. Prioritize candidates through documented deterministic criteria, not opaque LLM confidence.
18. Generate reports whose quantitative claims can be traced to source observations and computational results.
19. Preserve human oversight and avoid overstating scientific conclusions.

### 4.3 Product quality goals

20. Make the public image-exploration flow understandable to non-specialists while keeping technical details available to researchers.
21. Keep the codebase modular and testable without premature microservices or infrastructure.
22. Make unsupported, failed, unavailable, and insufficient-data outcomes visible rather than disguising them as success.

## 5. Non-Goals and Explicit Boundaries

The MVP will not:

- Claim that HELIOS-X has discovered or confirmed Planet X, a new planet, or any other new astronomical object.
- Replace astronomers, expert review, independent observations, or peer review.
- Treat a catalogue no-match as proof of novelty.
- Treat an image that visually appears to shift as confirmed physical motion before coordinate frames, registration, quality, and uncertainty are checked.
- Treat a successful empty archive query as a failed query, or a failed query as evidence that no object exists.
- Invent image frames, observation timestamps, coordinates, measurements, uncertainties, product identifiers, or catalogue matches.
- Treat the 102 spectral channels as 102 time epochs.
- Assume every sky position has repeated observations or a fixed six-month per-region cadence.
- Assume every product or channel is available for every region, date, or target.
- Promise continuous real-time monitoring of the entire sky or ingest and process more than a billion objects in the MVP.
- Automatically scan the entire SPHEREx archive for every possible anomaly in the MVP.
- Infer orbital parameters when observation count, temporal baseline, precision, or geometry do not support the calculation.
- Run AI agents merely to display or play images.
- Require users to create an investigation to use the public Explorer.
- Implement scientifically invalid image addition, subtraction, stacking, or cross-release comparison merely to expose a UI control.
- Introduce Kubernetes, Kafka, distributed microservices, or other infrastructure without a demonstrated requirement.
- Train a foundation model or require a custom LLM.
- Claim a feature is implemented simply because it appears in a specification.

## 6. Target Users

### 6.1 Primary users

**Public sky explorers and astronomy enthusiasts**

People who want to enter coordinates, view a region of the sky at different observation times, and visually inspect possible changes without needing to understand the archive's internal data structures.

**Astronomy students and researchers**

Users who want to inspect observations, evaluate source movement or variability, understand data-quality limitations, and conduct a more structured investigation.

**Scientific data analysts**

Users who need validated metadata, reproducible measurements, channel/product information, quality flags, catalogue associations, and provenance.

### 6.2 Secondary users

**Research reviewers and domain experts**

Users who inspect candidate assessments, evidence lineage, alternative explanations, and reports, and record review decisions when authorized.

**Science educators and communicators**

Users who demonstrate how repeated astronomical observations can reveal changes while clearly distinguishing visual indications from verified scientific measurements.

**Project operators and maintainers**

Users who monitor archive integration, data-release support, cache health, system errors, and deployment capability declarations.

## 7. Core User Journeys

### 7.1 Journey A — Explore a sky region (must work independently)

1. User opens the public Sky Image Explorer.
2. User enters RA and Dec, either in supported decimal-degree or sexagesimal form, and the UI makes the expected units and coordinate convention explicit.
3. User selects a supported FOV. Optional time-range, product-release, and spectral-channel filters are shown only when implemented and supported.
4. User submits the query.
5. The backend validates the query and discovers available SPHEREx observations through the configured adapter.
6. The UI shows the actual observation coverage and retrieval state.
7. The user opens the available image sequence.
8. The user plays, pauses, steps through, or selects frames on a timeline. A blink/rapid-toggle comparison may be offered when at least two compatible frames exist.
9. The user can inspect each frame's actual observation time, product/release, channel/wavelength, and available quality metadata.
10. If the archive returns no observations, one observation, incompatible products, or an error, the UI explains the actual outcome and does not invent additional frames.

### 7.2 Journey B — Investigate a region from the Explorer

1. User explores a region and selects **Investigate this region** or an equivalent action.
2. The application passes stable region parameters and observation/product/frame references—not large image arrays or client-invented measurements.
3. The backend creates an investigation and execution using the documented API contract.
4. The Workflow Orchestrator revalidates the input references, product compatibility, temporal coverage, quality, and analysis prerequisites.
5. Supported deterministic image/scientific functions execute.
6. Relevant agents interpret structured results, identify evidence gaps, and recommend additional supported checks.
7. The application stores results, provenance, task state, candidate assessments, and report versions.
8. The user returns to the Explorer or opens the investigation workspace and can navigate back to the relevant region and observations.

### 7.3 Journey C — Investigate a known target

1. User searches by a supported object name or enters its coordinates.
2. The application resolves the target through a documented name-resolution/catalogue source where available.
3. The resolved coordinates, reference frame, epoch, proper-motion treatment, and source of the resolution are shown or retained for inspection.
4. The application queries the actual SPHEREx observation coverage.
5. The user explores the returned frames and may start a scientific investigation if supported.
6. The system reports insufficient coverage or unsupported analysis when required; it does not pretend that every target has a useful multi-epoch sequence.

### 7.4 Journey D — Review an investigation

1. An authorized user opens an investigation.
2. The user monitors tasks, agent activity, retries, errors, and overall execution status.
3. The user inspects observations, measurements, quality checks, catalogue matches, evidence, and uncertainty.
4. The user reviews alternative explanations and the deterministic candidate-assessment rationale.
5. The user opens a report and traces quantitative claims to supporting evidence.
6. An authorized reviewer may record a decision or request further analysis where the feature is enabled.

## 8. Functional Requirements

Requirements are grouped by priority. P0 requirements define the NASA-aligned Explorer and the bounded integrated investigation MVP. P1 items are valuable enhancements but must not delay the core coordinate-to-image-sequence path. P2 items are later research or scale extensions.

### 8.1 P0 — Public Sky Image Explorer

#### EX-P0-01 — Public Explorer access

The application must provide a public route into the Sky Image Explorer. Viewing available public SPHEREx imagery must not require the visitor to create an investigation or invoke an AI agent. Public access must be subject to appropriate query, download, and rate limits.

#### EX-P0-02 — Coordinate input

The user must be able to specify a sky position using RA and Dec.

- Accept decimal degrees and sexagesimal notation when supported by the input parser.
- Label the expected units and format clearly.
- Normalize validated values into a canonical internal representation.
- Define and document the coordinate frame/reference assumptions; do not silently assume an epoch or proper-motion correction for a named moving target.
- Reject non-finite or out-of-range values with actionable validation messages.
- Support the NASA-provided WISE 0855 coordinate string: RA `08h55m10.831s`, Dec `-7d14m42.53s`.

#### EX-P0-03 — Sky-region and FOV selection

The user must be able to select a supported field of view around the input position. FOV units, geometry, minimum, maximum, and server-side limits must be explicit. The service must prevent requests that exceed documented resource limits.

#### EX-P0-04 — Observation discovery

For a valid region query, the system must discover available SPHEREx observations/products through the configured data-source adapter. It must return actual source records and clearly distinguish:

- successful query with multiple observations;
- successful query with one observation;
- successful query with zero observations;
- data product unavailable for the requested region or constraints;
- unsupported query option;
- archive timeout, provider error, malformed response, or other retrieval failure.

A zero-result query must not create a fabricated frame. A failed query must not be displayed as a successful empty result.

#### EX-P0-05 — Image retrieval and cutout display

The system must retrieve or access the image products needed to display the selected sky region. It must produce an appropriate cutout or browser-displayable preview when supported by the product and archive route.

- Preserve the original source product separately from any resized, compressed, rendered, or cutout derivative.
- Keep image pixels and large FITS files outside ordinary relational rows and ordinary JSON payloads.
- Retain a controlled reference to the original product and each generated derivative.
- A display preview must not be treated as the authoritative scientific measurement input when its rendering is lossy or otherwise unsuitable.
- The frontend must display actual returned imagery or a clearly labelled fixture; it must not use decorative/generated sky images as scientific observations.

#### EX-P0-06 — Time-ordered frame sequence

The backend must provide a sequence/manifest of available image frames. The frontend must order frames by the actual validated acquisition/observation timestamp.

Each frame must include, where available:

- stable observation, product, and frame references;
- actual observation/acquisition timestamp and time convention;
- mission, instrument, product type, data release, and processing version;
- channel/wavelength and detector/array identity when supplied by the product;
- dimensions, pixel scale, units, and WCS metadata or a validated WCS reference;
- quality, contamination, mask, variance, PSF, and calibration flags/references where available;
- controlled artifact/preview reference;
- retrieval status, source reference, and checksum if actually computed.

Observation time must remain distinct from retrieval time, release time, file timestamp, and spectral-channel identity. Missing metadata must be represented as unavailable rather than fabricated.

#### EX-P0-07 — Video-like image viewer

The frontend must support:

- play and pause;
- previous and next frame;
- a timeline or frame selector;
- display of the selected frame index and actual observation time;
- a documented playback interval or speed control when applicable;
- clear behavior at the first and last available frame;
- a loading state while a frame is being fetched;
- a truthful empty/single-frame state when playback is not possible.

The viewer must not interpolate or synthesize intermediate observations. Playback simply presents available actual frames in sequence.

#### EX-P0-08 — Visual frame comparison

When at least two compatible frames are available, the user must be able to compare them through frame stepping and/or a blink/rapid-toggle interaction. Any comparison must clearly identify the frames and timestamps being compared.

A simple visual blink between unregistered frames may help a person notice apparent changes, but the application must not label that apparent shift as measured physical motion. Scientific motion claims require the registration, source association, uncertainty, and quality conditions defined in `SCIENTIFIC_METHODOLOGY.md`.

#### EX-P0-09 — Frame metadata and quality visibility

The user must be able to inspect the metadata available for the current frame, including actual observation time, product/release, channel/wavelength, units, WCS information, and quality/contamination flags where supplied. The UI must distinguish unavailable metadata from a clean or passed quality result.

#### EX-P0-10 — Data-release updateability

The data-source integration must isolate archive-specific details behind a replaceable adapter. Release availability must be checked against current authoritative archive documentation and/or the archive response; the application must not assume a release, product, region, channel, or time range is always available.

Use the release-selection policy in `DATA_SOURCE_SPEC.md`. The current project data-source specification prefers QR3 where available and appropriate, permits QR2 under its documented conditions, and prohibits silent cross-release mixing. Revalidate that policy during implementation against current official documentation.

#### EX-P0-11 — Graceful coverage and capability behavior

The Explorer must explain when:

- no usable observations are available;
- only one frame is available;
- observations exist but products are incompatible for a requested comparison;
- a preview cannot be generated;
- required metadata or WCS is missing;
- an optional operation is not supported;
- archive retrieval fails or is still running.

The system must not turn any of these states into fabricated successful playback or scientific output.

#### EX-P0-12 — NASA target validation

The Explorer must include reproducible target test scenarios for:

1. Barnard's Star.
2. Epsilon Indi B.
3. WISE 0855, using the challenge-provided coordinates above.
4. RW Aurigae.
5. R Draconis.

For each target, tests must validate name/coordinate resolution where implemented, query construction, actual archive-coverage discovery, frame manifest correctness, and honest handling of zero/one/multiple usable observations. The tests must not require a fabricated positive detection or assume that every named target has a sufficient multi-epoch baseline. Target coordinates not supplied in the challenge must be resolved from an authoritative, recorded source; proper motion and reference epoch must be handled as appropriate.

### 8.2 P0 — Agentic Scientific Investigation Engine

#### AI-P0-01 — Investigation creation

The system must create an investigation record containing a unique identifier, objective, input sky region or target reference, selected observation/product references, data-source configuration, supported analysis options, relevant parameters, configuration/workflow versions, creation time, and status.

#### AI-P0-02 — Explorer-to-investigation handoff

A user must be able to explicitly start an investigation from a selected Explorer region or observation sequence. The handoff must use stable references and validated parameters. It must not send large image arrays through the investigation-creation request or trust measurements supplied by the browser.

Opening, searching, or playing images must not create an investigation implicitly.

#### AI-P0-03 — Capability and prerequisite validation

Before each analysis, the system must evaluate whether the chosen product, observations, temporal baseline, metadata, uncertainty information, and implemented method support that analysis. Use distinct outcomes such as `NOT_SUPPORTED`, `INSUFFICIENT_DATA`, and `BLOCKED` according to the canonical contracts.

#### AI-P0-04 — Workflow orchestration

A deterministic Workflow Orchestrator must validate plans, manage dependencies, execute tasks in valid order, record task states, apply bounded retries and timeouts, support cancellation where enabled, enforce mandatory evidence gates, and preserve partial failures. Mandatory workflow policy must not be controlled by an LLM.

#### AI-P0-05 — Specialized agent roles

Preserve the eight logical AI-agent roles defined in `AGENT_SPECIFICATIONS.md`:

1. Investigation Planner Agent.
2. Data Retrieval Agent.
3. Data Quality Agent.
4. Change Detection Agent.
5. Motion Analysis Agent.
6. Spectral & Photometry Agent.
7. Catalogue Matching Agent.
8. Validation & Hypothesis Agent.

These are logical responsibilities, not a requirement for eight separately deployed LLM processes. Invoke only the roles relevant to the investigation and available capabilities. An agent may interpret validated structured outputs and request approved operations; it may not fabricate numerical science or bypass gates.

#### AI-P0-06 — Deterministic scientific processing

The MVP must implement at least one complete, useful scientific analysis supported by verified data. Candidate methods include WCS/coordinate validation, image registration, source detection, source association, measured positional change, photometry, variability checks, or spectral measurements. Exact MVP methods must be selected through data feasibility and documented in `SCIENTIFIC_METHODOLOGY.md`.

Every implemented method must define inputs, units, coordinate/time conventions, algorithm version, assumptions, uncertainty handling, quality conditions, failure statuses, limitations, and known-answer validation tests. An LLM must not be required to calculate numerical results.

#### AI-P0-07 — Data quality and alternative explanations

The system must implement relevant checks for available data, which may include image registration quality, source blending/contamination, background/noise, signal-to-noise, variance, calibration/PSF information, quality flags, source-association ambiguity, known-object matches, and inconsistent measurements. The report must disclose which checks were performed and which were unavailable or not applicable.

#### AI-P0-08 — Catalogue matching

When a suitable external catalogue and access method are available, record the catalogue name/release, query, source identifiers, coordinates, matching criteria, angular separation or relevant metric, uncertainty, competing matches, and retrieval status. Catalogue matching must be capability-aware and must not treat a no-match as proof of novelty.

#### AI-P0-09 — Evidence and provenance

Persist applicable source observation/product references, query and retrieval metadata, actual observation timestamps, product releases, image artifact references, method and software versions, parameters, measurements, units, uncertainty, quality outcomes, agent run metadata, warnings, failures, and related investigation/candidate identifiers.

Observed data, derived measurements, quality outcomes, agent interpretations, hypotheses, candidate assessments, and human decisions must remain distinct.

#### AI-P0-10 — Candidate assessment

Candidate prioritization must use documented, versioned deterministic criteria applied to validated evidence. Candidate scores or statuses must be explainable and must not be derived solely from LLM confidence. `PRIORITIZED_FOR_REVIEW` means eligible for human attention, not scientific confirmation.

#### AI-P0-11 — Scientific reports

Generate reports from persisted, validated records. Include the investigation objective, selected data/release, actual observation times, methods and versions, supported measurements and uncertainty, quality and evidence-gate outcomes, catalogue associations, supporting and contradictory evidence, alternative explanations, unavailable checks, limitations, candidate status/rationale, and recommended next steps when justified.

Every quantitative report claim must link to a stored authoritative result. LLM-assisted prose may improve readability but must not add facts absent from the records.

#### AI-P0-12 — Human oversight

Where human review is enabled, authorized reviewers may accept a candidate for further investigation, reject it with a reason, defer a decision, or request re-analysis. Record reviewer, timestamp, rationale, and the evidence/result version reviewed. A human decision must not silently modify source observations or scientific measurements.

#### AI-P0-13 — Failure and history handling

The system must distinguish `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRYING`, `CANCELLED`, `BLOCKED`, `NOT_SUPPORTED`, and `INSUFFICIENT_DATA` task states, along with separate quality, evidence-gate, and candidate outcomes. Preserve execution history; retries or new analysis configurations must not silently overwrite previous results.

### 8.3 P0 — Shared platform requirements

#### PL-P0-01 — Shared data foundation

The Explorer and investigation engine must use the same validated data-source adapters, normalized observation/product metadata, artifact references, coordinate conventions, and provenance rules. The two product capabilities must not maintain competing definitions of observation time, release, channel, detector array, or quality state.

#### PL-P0-02 — Controlled image artifacts

Large FITS files and image arrays must be stored outside relational database rows and delivered through controlled artifact access. The browser must not receive arbitrary filesystem paths. Generated preview/cutout derivatives must retain links to their original product and processing configuration.

#### PL-P0-03 — Public/private boundary

Public users may query and view public mission imagery through rate-limited read-only Explorer capabilities. Private saved investigations, reviewer actions, and protected resources must use backend-enforced access control as defined in `SECURITY.md`. Public access must never imply unrestricted access to server files or private investigation records.

#### PL-P0-04 — Honest capability disclosure

The application must declare which mission products, releases, bands/channels, comparison methods, catalogues, and scientific analyses are supported by the current deployment. A documented future feature must not be represented as an active capability.

### 8.4 P1 — Recommended enhancements

These correspond to optional considerations in the NASA challenge or useful extensions to the investigation experience. They should be implemented only after the P0 Explorer and investigation paths are reliable.

1. **Spectral-channel exploration:** let users select and compare available SPHEREx spectral channels/wavelength samples. The UI must make clear that channel selection is spectral, not temporal.
2. **Detector-array combination:** where the documented products expose the six detector arrays and the combination is scientifically valid, allow array-specific inspection and validated combination to improve signal-to-noise. Do not assume six arrays are six epochs or blindly average incompatible data.
3. **On-demand spectrum generation:** generate a spectrum for a selected source/region only when required calibrated inputs, channel metadata, uncertainty, and extraction method are available.
4. **Image subtraction:** generate difference images only after validating compatible products, WCS, pixel grid, photometric scale, background, masks, PSF/resolution, and registration quality. Label the method and limitations.
5. **Image addition/stacking:** combine images only using a documented scientifically valid method with compatible calibration and uncertainty treatment. Distinguish sum, mean, weighted stack, and other operations by their actual implementation.
6. **Gaia and WISE overlays:** retrieve supported external catalogue data and display matched sources with catalogue identity, release, matching radius, coordinate frame/epoch, and separation. Overlay presentation must not imply a match is exact when it is ambiguous.
7. **Shareable coordinate URLs:** encode validated query state such as RA, Dec, FOV, time range, and selected channel in a stable URL so a sky region can be opened or embedded elsewhere. Do not place secrets or private investigation data in URLs.
8. **More advanced motion and variability visualization:** show measured tracks, flux histories, uncertainties, and quality information only when supported by validated observations and methods.
9. **Optional saved regions and user preferences:** add only with a clear data-retention and access-control policy.

### 8.5 P2 — Research and scale extensions

- Broader archive and mission integrations.
- Large-scale batch source screening over substantial archive areas.
- Advanced spectral interpretation and model fitting.
- Orbit fitting when observation count, precision, temporal baseline, and geometry justify it.
- Bayesian model comparison with documented priors and likelihoods.
- Distributed processing or scalable job infrastructure when measured workload requires it.
- Collaboration, annotations, and advanced research workspaces.
- Additional automated anomaly-ranking and triage methods with calibrated, validated performance.

These are not MVP commitments.

## 9. Scientific and Data Integrity Requirements

### 9.1 Time, spectral channel, and detector array are different dimensions

- **Observation epoch/time:** when the source was actually observed; this determines the time ordering of image frames.
- **Spectral channel/wavelength:** the wavelength sampling dimension; SPHEREx's 102 channels must never be treated as 102 time samples.
- **Detector array:** an instrument/data-product dimension. Array identity must be preserved and must not be mistaken for observation time or wavelength without product documentation.
- **Retrieval time:** when HELIOS-X requested/downloaded a product; it is not the observation time.
- **Release/processing version:** identifies a product version; it is not an observing epoch.

### 9.2 Observation cadence and coverage

The NASA challenge describes an approximately six-month all-sky mapping cadence. HELIOS-X must use the actual observation timestamps and regional coverage in the selected archive products. It must not interpolate or manufacture missing epochs, assume uniform cadence, or state that each coordinate is revisited exactly every six months.

### 9.3 Coordinate and target integrity

- Normalize validated sky positions to the documented internal coordinate convention.
- Retain the original user input and the normalized coordinates.
- Record coordinate frame, reference epoch, and proper-motion treatment when applicable.
- Named-object resolution must record its authoritative source and the epoch used.
- The application must not silently move a target coordinate between epochs without a documented method.

### 9.4 Image comparison integrity

Before any scientifically interpreted frame comparison, validate product compatibility, WCS, coordinate frame, pixel scale, units, channel/wavelength, calibration/release, masks, quality flags, and actual acquisition times. Image registration must have explicit residual and acceptance criteria appropriate to the intended method. If registration fails, dependent motion claims must be blocked or qualified.

A visual blink is an exploratory aid; it is not itself a scientific motion measurement.

### 9.5 Photometry and variability integrity

Brightness changes must account for compatible measurement definitions, calibration, uncertainty, detector behavior, background, contamination, and quality flags. A variable-star label or visible brightness variation must not be treated as a validated variability measurement until the selected data and method support it.

### 9.6 Candidate and discovery language

Use language such as *possible moving source*, *candidate*, *measured positional change*, *unmatched in the queried catalogue*, or *prioritized for review* only when the corresponding method and evidence justify it. Never call a candidate a newly discovered planet or confirmed Planet X solely because an automated system ranks it.

### 9.7 Missing and failed evidence

Missing data is not zero. An unavailable quality flag is not a passed quality check. A failed archive request is not a successful no-match. Unknown or inconclusive evidence gates must not silently pass.

## 10. Data Source Strategy

### 10.1 Primary source

NASA/IPAC IRSA is the intended primary archive for SPHEREx image/spectral-image products. The SPHEREx adapter must be replaceable and must isolate archive-specific protocols from the rest of HELIOS-X.

### 10.2 Product and release policy

Use `DATA_SOURCE_SPEC.md` as the authority for product identification, current release policy, access routes, product semantics, and metadata. The current project specification prefers QR3 when available and scientifically appropriate, allows QR2 only under its documented conditions, and prohibits silent mixing of releases. QR1 must not be treated as current if it is documented as retired. These statements must be rechecked against current official archive documentation during implementation; this PRD does not independently guarantee archive availability.

### 10.3 Retrieval behavior

The product must:

- query current archive availability rather than assuming all requested data exist;
- preserve exact query parameters and actual retrieval timestamps;
- distinguish zero results, unavailable products, unsupported filters, timeouts, malformed responses, and provider failures;
- validate downloaded product structure, size, WCS, units, timestamps, channel metadata, calibration, and quality information as applicable;
- preserve source IDs, release, processing versions, and provenance;
- use controlled caching where valid and define invalidation when release/source artifacts change;
- avoid repeatedly downloading the same immutable artifact when a safe, valid cache entry exists.

### 10.4 Auxiliary catalogues

Gaia and WISE are optional P1 integrations. They are not primary replacements for SPHEREx. Their use must be governed by `DATA_SOURCE_SPEC.md`, the scientific matching method, and documented release/epoch handling.

### 10.5 Large archive scope

The challenge's billion-object scale motivates on-demand search and efficient retrieval. The MVP is not required to download, index, or process every SPHEREx image/object. It must retrieve the smallest appropriate products/cutouts for a user-requested region, apply safe resource limits, and preserve the ability to extend to larger workloads later.

## 11. Product Information Architecture

### 11.1 Primary navigation

- **Explore Sky** — public coordinate search and image-sequence viewer.
- **Investigations** — saved investigations and their execution status, subject to access control.
- **Capabilities** — supported missions, releases, products, channels, analysis methods, and known limitations.
- **Help / About** — coordinate input help, data interpretation guidance, and mission/source attribution.
- **Settings** — only real settings supported by the implementation, such as display preferences.

### 11.2 Explorer screen

The Explorer screen should provide:

- RA and Dec input with examples, format guidance, and validation.
- FOV selection with explicit units and supported bounds.
- Optional time range and available product/channel filters when implemented.
- Observation coverage summary and retrieval state.
- Main image display with zoom/pan if supported by the chosen viewer implementation.
- Frame timeline and play/pause/step controls.
- A visible observation-time label for the active frame.
- Frame metadata and quality/contamination information.
- An explicit action to investigate the selected region.
- Clear empty, loading, partial, unsupported, and failed states.

### 11.3 Investigation workspace

The investigation workspace should provide:

- investigation objective and selected region/source;
- selected observation/product references;
- execution and task monitor;
- relevant agent roles and their status;
- scientific measurements, units, uncertainties, and quality outcomes;
- evidence and provenance navigation;
- candidate assessment and rationale;
- report viewer and review state.

### 11.4 Visual and interaction principles

- The Explorer is the first and simplest product experience.
- Evidence and scientific state take precedence over decorative effects.
- Use a dark-first, high-contrast visual system with a deliberate light theme, consistent with `UI_UX_SPECIFICATION.md`.
- Do not show fake mission telemetry, fabricated images, decorative scientific charts, invented progress percentages, or nonfunctional controls.
- Use accessible labels, keyboard-operable controls, meaningful empty/error states, and responsive layouts.
- Present public-facing explanations first and expose technical details progressively.

## 12. Product Data and Conceptual Objects

The product must work with the resource concepts governed by the API and database specifications. At the product level, the important concepts are:

- **Sky Query:** normalized RA/Dec, coordinate frame/reference metadata, FOV geometry, optional time/channel/release constraints.
- **Observation:** a real source observation with actual acquisition time and source identifier where available.
- **Product:** a documented data product and release/processing version.
- **Image Artifact:** original product reference or generated cutout/preview, dimensions, format, checksum when computed, and controlled access reference.
- **Frame:** an image representation associated with a specific observation/product/time and its metadata.
- **Frame Sequence:** an ordered collection of available frames for one query; it may contain zero, one, or multiple frames.
- **Investigation:** a bounded scientific objective, configuration, and input references.
- **Execution and Task:** a run of the investigation and its individual workflow units.
- **Measurement:** a deterministic scientific result with method, units, uncertainty, and input provenance.
- **Evidence:** a traceable observation, artifact, measurement, quality result, or other supported record.
- **Candidate:** a source under investigation, not a confirmed discovery.
- **Candidate Assessment:** a versioned evaluation using deterministic documented criteria.
- **Report and Review:** traceable summary and optional human decision records.

The PRD does not define exact table fields or endpoint names. Those belong in `DATABASE_SCHEMA.md` and `API_SPECIFICATION.md` respectively.

## 13. Scientific Outputs and Reporting

### 13.1 Explorer outputs

For each query, the Explorer should show:

- normalized query coordinates and FOV;
- data source and current supported product/release information;
- actual observation count and observation times;
- frame availability and retrieval status;
- available image metadata and quality flags;
- unsupported or unavailable fields/operations;
- provenance references sufficient to identify the source products.

### 13.2 Investigation outputs

Where supported by the selected data and methods, the report should include:

1. Investigation objective and scope.
2. Sky region/target and coordinate reference information.
3. Data sources, product releases, observation IDs, and actual observation timestamps.
4. Retrieval outcomes and data limitations.
5. Methods, software versions, parameters, and assumptions.
6. Image-registration and source-association outcomes when used.
7. Measurements, units, uncertainties, and quality outcomes.
8. Spectral/photometric information where supported.
9. Catalogue associations, competing matches, and match criteria.
10. Supporting and contradictory evidence.
11. Alternative explanations considered and checks not performed.
12. Evidence-gate outcomes and candidate status/rationale.
13. Missing evidence, failures, warnings, and limitations.
14. Recommended follow-up when justified.
15. Human-review state and report version, where available.

The system must omit unsupported quantities or mark them unavailable. It must not fill missing fields with plausible-looking generated values.

## 14. Non-Functional Requirements

### 14.1 Scientific integrity

All quantitative scientific results must originate from named, versioned deterministic methods or documented source data. All methods must define preconditions, units, uncertainties, tolerances, failure modes, and limitations.

### 14.2 Performance and resource control

- Use on-demand regional queries and cutouts rather than downloading the full archive for ordinary browsing.
- Keep image artifacts outside JSON response bodies and relational rows.
- Bound FOV, frame count, time range, file size, processing duration, concurrent retrievals, and public request rates.
- Use caching only with keys that include the relevant query, product, release, processing version, and rendering parameters.
- Define performance baselines and release targets from representative data and measured environments; do not claim arbitrary latency guarantees without testing.

### 14.3 Reliability

- Distinguish liveness, readiness, archive availability, and scientific capability.
- Record retries and failures.
- Preserve completed results when optional downstream components fail.
- Do not let LLM provider outages corrupt or block deterministic science that can otherwise run.
- Document the limits of in-process execution and restart recovery.

### 14.4 Security and public access

- Public Explorer endpoints must be read-only with respect to mission data and protected by rate, size, and resource limits.
- Validate coordinates, time ranges, FOV, filters, file references, and all external data.
- Do not expose arbitrary filesystem paths or allow user-provided paths to be opened by the server.
- Treat archive metadata, FITS headers, catalogue values, and external text as untrusted data, never as instructions.
- Keep secrets outside code and browser bundles.
- Enforce backend authorization for private investigations, execution controls, reviews, and operational endpoints.
- Use HTTPS and explicit CORS configuration in shared/public deployments.
- Follow `SECURITY.md` for final policy and implementation details.

### 14.5 Accessibility and usability

- Coordinate entry and image controls must be keyboard accessible.
- Controls must have labels and understandable validation messages.
- Do not rely only on color to communicate quality or execution state.
- Support responsive desktop, tablet, and mobile layouts.
- Provide text descriptions for image metadata and a usable non-animation way to select frames.
- Respect reduced-motion preferences where applicable.

### 14.6 Maintainability and extensibility

- Use a modular monolith for the MVP.
- Keep archive-specific logic behind adapter interfaces.
- Separate API contracts, persistence models, deterministic science, agents, and presentation.
- Make product releases and new source products configurable through documented adapters and metadata, not hardcoded assumptions.
- Add dependencies only when tied to a concrete capability and verified use case.

### 14.7 Observability and auditability

Record correlation/request IDs, query parameters, source/release, artifact references, task transitions, method/agent versions, durations, errors, and safe diagnostics. Do not log secrets, full image arrays, or large sensitive payloads by default.

## 15. Product Metrics

Metrics must be evaluated using documented test fixtures, actual archive-derived cases where available, and repeatable measurement procedures. Targets must be set before a formal release evaluation.

### 15.1 Explorer usability and correctness

- Coordinate validation correctness across decimal and supported sexagesimal inputs.
- Percentage of test queries returning correct zero/one/multiple-frame states.
- Percentage of returned frame manifests correctly ordered by actual observation time.
- Percentage of displayed frame metadata that matches the source product.
- Successful completion rate of the coordinate-to-image-sequence journey on supported test regions.
- Number of fabricated or incorrectly labelled frames (target: zero).

### 15.2 Scientific reliability

- Detection rate for known data-quality failure cases.
- Numerical agreement with known-answer fixtures within method-specific tolerances.
- Completeness of uncertainty and provenance fields where applicable.
- Correct handling of unsupported and insufficient-data cases.
- Rate of unsupported scientific claims in generated reports (target: zero).

### 15.3 Investigation reliability

- Percentage of runs with explicit task outcomes and appropriate error records.
- Percentage of quantitative report claims linked to authoritative stored results.
- Reproducibility of fixed-input, fixed-method investigations within documented tolerances.
- Correct handling of retries, cancellation, and historical result versions.

### 15.4 Operational efficiency

- Query, cutout, and frame-delivery latency under a documented representative environment.
- Cache hit ratio for repeat requests where caching is valid.
- Data transfer and processing volume per Explorer query.
- Failure and retry rates by source/release.

The number of claimed discoveries must not be used as an MVP success metric.

## 16. Acceptance Criteria and Release Gates

A specification is not evidence of implementation. Each criterion must be supported by an executable test, recorded verification, or an explicit `NOT RUN`/`BLOCKED` status.

### 16.1 Explorer acceptance gate — required for NASA challenge readiness

The Explorer is acceptable only when:

1. A public user can open the Explorer without creating an investigation.
2. RA and Dec inputs are validated and normalized correctly.
3. FOV input is validated against documented limits.
4. A supported query retrieves actual SPHEREx observation records from the verified adapter or uses a clearly labelled fixture during development.
5. The system returns zero, one, or multiple frames truthfully.
6. Multiple frames are ordered by actual acquisition time.
7. The frontend displays the actual image cutouts/previews returned by the backend.
8. The user can play, pause, step, and select frames on a timeline.
9. The active frame's actual observation timestamp is visible.
10. Product/release/channel and available quality metadata can be inspected.
11. The application never presents generated or synthetic frames as live SPHEREx observations.
12. Query failures, no coverage, corrupt data, unsupported operations, and single-frame results have distinct, understandable UI states.
13. The five NASA targets have reproducible validation scenarios and honest archive-coverage outcomes.
14. Public query, artifact delivery, and resource limits are enforced.

**Live-data requirement:** fixture-based development is acceptable, but a NASA challenge demonstration must clearly distinguish fixture behavior from archive-derived behavior. Any claim that live SPHEREx data are supported requires a verified official archive access route and tested real products.

### 16.2 Agentic investigation acceptance gate

The investigation layer is acceptable when:

1. A user can start a bounded investigation from valid Explorer context or another documented input.
2. The backend creates an investigation and execution with stable IDs and recorded inputs.
3. Capability and prerequisite checks occur before scientific analysis.
4. At least one useful deterministic scientific method is implemented and passes known-answer tests.
5. Relevant agent roles receive typed inputs and return validated structured outputs.
6. Agent failure cannot fabricate or corrupt measurements/evidence.
7. Evidence and provenance can be retrieved and traced.
8. Candidate assessment uses deterministic, versioned criteria.
9. A report can be generated from persisted validated records.
10. Unsupported, insufficient-data, blocked, failed, cancelled, and successful-empty outcomes remain distinct.
11. Historical results are preserved after retries or re-analysis.
12. Human review is recorded when enabled and does not alter raw scientific observations.

### 16.3 Integrated product acceptance gate

The combined product is acceptable when:

1. The Explorer works independently of the agentic engine.
2. A user can explicitly promote a selected region/observation sequence into an investigation.
3. The handoff preserves exact query, observation, product, and frame references.
4. The investigation revalidates the inputs and does not trust client-provided measurements.
5. Investigation results link back to the region and observations shown in the Explorer.
6. Reports preserve provenance from each quantitative claim to input products and methods.
7. The two journeys share consistent coordinate, timestamp, release, channel, detector, quality, and artifact semantics.

### 16.4 NASA target test matrix

| Target | Challenge relevance | Required validation behavior |
|---|---|---|
| Barnard's Star | Nearby high-proper-motion star | Resolve the target from an authoritative coordinate source; record epoch/proper-motion handling; inspect actual available frames; test whether positional comparison is supported. |
| Epsilon Indi B | Brown-dwarf pair associated with a star | Resolve coordinates and inspect whether the selected product's resolution/blending supports separate-source interpretation; do not assume both components are resolved. |
| WISE 0855 | Fast-moving brown dwarf | Use challenge-supplied RA/Dec; validate sexagesimal parsing, query coverage, frame sequence, and whether motion analysis prerequisites are met. |
| RW Aurigae | Variable star | Inspect actual observation times and photometric suitability; test variability only if adequate compatible data and uncertainty information exist. |
| R Draconis | Variable star | Inspect actual observation times and photometric suitability; test variability only if adequate compatible data and uncertainty information exist. |

The test result for each target may be supported, insufficient-data, unsupported, zero coverage, or a retrieval failure. Passing the target test means the application behaved correctly for the verified result; it does not require a positive detection.

### 16.5 Overall release gate

Do not declare the combined MVP ready until the applicable Explorer, investigation, integration, scientific, security, and operational acceptance criteria pass. Any fixture-only or unverified live-data capability must be labelled accurately.

## 17. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Product becomes an agent dashboard rather than the NASA-requested Explorer | Make the public Sky Image Explorer the primary entry point and release gate; keep investigation as an explicit second layer. |
| Explorer playback is mistaken for scientific motion analysis | Clearly distinguish visual comparison from registered, uncertainty-aware measured motion. |
| A sky region has no repeated observations | Show actual zero/one/multiple coverage; never synthesize frames or promise fixed cadence. |
| 102 spectral channels are confused with epochs | Model observation time, channel/wavelength, and detector array as separate dimensions throughout the product. |
| Product releases or processing versions are mixed | Use release-aware adapters and compatibility checks; never mix silently. |
| Images are too large for browser/API handling | Use bounded cutouts, previews, controlled artifact delivery, caching, and size limits. |
| Image subtraction/addition produces misleading artifacts | Require compatibility, registration, calibration, PSF/background, masks, and uncertainty checks before enabling. |
| Detector-array combination is scientifically invalid | Preserve array identity and implement combination only when product documentation and validation support it. |
| A named target cannot be resolved or has insufficient coverage | Record coordinate source/epoch; return explicit unsupported or insufficient-data outcomes. |
| Agents generate unsupported claims | Require evidence-linked structured outputs and deterministic report validation. |
| Catalogue no-match is mistaken for novelty | Use explicit unmatched terminology and alternative-hypothesis checks. |
| Public endpoints are abused or overload the archive | Enforce FOV/frame/file limits, rate limits, caching, timeouts, and controlled concurrency. |
| Requirements diverge across documents | Treat this PRD as product authority and update API, database, UI, data-source, testing, implementation, and master prompt contracts in the declared creation order. |
| Project scope expands excessively | Complete the P0 coordinate-to-image path and one bounded Explorer-to-investigation path before P1/P2 extensions. |

## 18. Decisions and Conflict Resolutions

### 18.1 Primary product versus agentic system

**Conflict:** The earlier PRD positioned HELIOS-X primarily as an Agentic Scientific Investigation System. The NASA challenge explicitly requires a public-facing image tool that displays sky images over time.

**Resolution:** HELIOS-X is a two-capability integrated platform. The Sky Image Explorer is the primary public experience and NASA challenge deliverable. The Agentic Scientific Investigation Engine remains a core product capability and differentiator, but is accessed explicitly after or alongside exploration. This preserves the earlier agentic plan without allowing it to replace the NASA-required viewer.

### 18.2 Image playback versus automated motion detection

**Conflict:** The NASA brief requires users to see a time-ordered sequence; the earlier plan emphasizes scientific source tracking and measurement.

**Resolution:** Treat visual frame playback/blinking as a P0 Explorer capability. Treat validated source detection, registration, motion estimation, and evidence-based interpretation as the scientific investigation layer. A blinking image alone is not a scientific motion measurement.

### 18.3 Public access versus protected investigations

**Conflict:** A public-facing tool should be easy to access, while private investigation resources and review actions require security controls.

**Resolution:** Public, read-only Explorer use must not require an account. Saved/private investigations, cancellation/retry controls, reviewer actions, and protected reports must use backend-enforced authorization. Public demo examples may be provided as clearly labelled read-only examples.

### 18.4 Mission cadence versus actual regional coverage

**Conflict:** The challenge describes an approximately six-month all-sky cadence; the application cannot assume that every region has a usable image at every nominal cycle.

**Resolution:** Use actual archive timestamps and per-region product availability. Display gaps and missing epochs rather than interpolating or fabricating frames.

### 18.5 Optional NASA features versus MVP scope

**Conflict:** The challenge lists image addition/subtraction, six-array combination, spectra, overlays, FOV, URL queries, and metadata as potential considerations, not all as mandatory.

**Resolution:** Include FOV selection and basic image metadata in P0 because they are directly useful to the primary Explorer journey. Place complex spectral, detector-array, image arithmetic, and auxiliary catalogue features in P1 until their source-product semantics and scientific acceptance tests are verified. This protects the core delivery schedule and scientific correctness.

### 18.6 Scientific automation versus human discovery

**Conflict:** The agentic system can rank unusual candidates, but automated ranking is not scientific confirmation.

**Resolution:** The product can identify possible changes, measure validated quantities, compare known catalogues, and prioritize candidates for review. Scientific confirmation remains with qualified humans and independent verification.

## 19. Dependencies and Related Documents

This PRD defines product goals, user journeys, priorities, boundaries, and acceptance criteria. The following documents define implementation contracts and must be kept consistent with this PRD:

- `SYSTEM_ARCHITECTURE.md` — component boundaries and shared Explorer/investigation data flow.
- `AGENT_SPECIFICATIONS.md` — eight logical agent roles, tool permissions, inputs, outputs, and failure behavior.
- `SCIENTIFIC_METHODOLOGY.md` — scientific assumptions, image registration, source motion, photometry, spectra, uncertainty, quality, and validation.
- `INVESTIGATION_WORKFLOW.md` — task ordering, dependencies, capability checks, and evidence gates.
- `DATA_SOURCE_SPEC.md` — SPHEREx/IRSA products, release policy, metadata, archive methods, and target validation.
- `API_SPECIFICATION.md` — exact public API routes, payloads, response schemas, statuses, and error formats.
- `DATABASE_SCHEMA.md` — persistence entities, frame/artifact references, relationships, constraints, and migrations.
- `BACKEND.md` — backend service responsibilities, retrieval, image processing, artifacts, orchestration, and integration.
- `FRONTEND.md` — frontend routes, components, viewer implementation, API integration, and states.
- `UI_UX_SPECIFICATION.md` — navigation, Explorer interactions, viewer behavior, visual design, and accessibility.
- `TESTING_AND_ACCEPTANCE.md` — Explorer, target, scientific, agent, API, and end-to-end acceptance tests.
- `SECURITY.md` — public access, authentication, authorization, artifact handling, and security controls.
- `OPERATIONS.md` — setup, data/artifact directories, migrations, logging, backup, and recovery.
- `IMPLEMENTATION_PLAN.md` — build sequence, phase gates, repository layout, and completion criteria.
- `MASTER_DEVELOPMENT_PROMPT.md` — controlling document-review and sequential implementation instructions.

### 19.1 Contract ownership

- This PRD controls product identity, user outcomes, priority, scope, and acceptance intent.
- `API_SPECIFICATION.md` controls exact HTTP behavior and payloads.
- `DATABASE_SCHEMA.md` controls persistence fields, relations, constraints, and migrations.
- `DATA_SOURCE_SPEC.md` and verified official product documentation control archive/product semantics and source availability.
- `SCIENTIFIC_METHODOLOGY.md` controls scientific methods, preconditions, uncertainty, and numerical interpretation.
- `INVESTIGATION_WORKFLOW.md` controls task order and evidence gates.
- `SECURITY.md` controls access and security policy.
- `FRONTEND.md` and `UI_UX_SPECIFICATION.md` control frontend behavior and presentation.

If a lower-level document conflicts with this PRD's product requirements, record the conflict and update the authoritative contract before implementing the affected feature. Do not silently create a second API, schema, or scientific rule.

## 20. Open Decisions Before the Affected Feature Is Built

The following items must be resolved in the owning documents before their implementation. They must not be guessed in code:

1. Exact SPHEREx product types and image extensions suitable for the first Explorer release.
2. Verified official archive query/download/cutout route and current supported release availability.
3. Which regions and target objects have usable distinct observation times in the selected products.
4. Exact coordinate frame/reference epoch and named-target resolution source for each target.
5. Supported FOV geometry, bounds, and maximum frame count for public queries.
6. Exact image preview/cutout format, WCS preservation, pixel scale, and controlled delivery mechanism.
7. The minimum compatible-image requirements for blinking comparison, subtraction, stacking, and detector-array combination.
8. Which spectral channels/wavelength metadata are available in the selected product and how an on-demand spectrum is derived.
9. Which catalogue services and releases are suitable for Gaia/WISE overlays and cross-matching.
10. Authentication policy and quotas for saved investigations and public-triggered computational jobs.
11. Which deterministic method constitutes the first end-to-end scientific investigation.
12. Reference fixtures, expected outputs, and method-specific numerical tolerances for the five challenge targets and synthetic tests.
13. Performance baselines for query, cutout retrieval, playback, and investigation execution in the intended deployment environment.

Each decision must record its chosen option, rationale, owner document, validation evidence, and remaining limitation.

---

## 21. MVP Boundary and Delivery Strategy

The MVP is a **working integrated product**, not a collection of disconnected mock screens. Its delivery must establish two demonstrable paths:

### Path A — NASA public Explorer

`RA/Dec + FOV → actual SPHEREx observation discovery → image cutouts/frames → time-ordered manifest → interactive playback/comparison → metadata/provenance`

### Path B — Agentic scientific investigation

`Selected Explorer region/observations → validated investigation request → capability/quality checks → deterministic analysis → relevant agent interpretation → evidence/candidate assessment → traceable report`

The paths share source adapters, coordinate conventions, product metadata, image artifacts, quality information, and provenance. Path A must remain usable if the LLM provider or agentic engine is unavailable. Path B must never trust unvalidated client-supplied scientific results.

### 21.1 Development sequence

The recommended product-level sequence is:

1. Reconcile the PRD and system contracts.
2. Verify the initial SPHEREx data product and archive access route; define honest fixture behavior.
3. Establish shared coordinate, observation, product, frame, artifact, and status contracts.
4. Implement the backend fixture adapter and observation-discovery contract.
5. Implement the coordinate/FOV Explorer query and frame manifest.
6. Implement controlled image artifact/cutout delivery and the frontend image-sequence viewer.
7. Validate the Explorer against the five NASA target scenarios and representative zero/one/multiple-frame cases.
8. Implement one deterministic scientific analysis supported by the selected products.
9. Implement Explorer-to-investigation handoff, orchestrator, evidence/provenance, and report path.
10. Integrate relevant agent roles under strict tool and schema controls.
11. Verify live IRSA/SPHEREx retrieval and update release capability declarations.
12. Add P1 features only after the P0 release gates pass.
13. Run combined end-to-end, security, operational, and scientific acceptance checks.

This is a product-level sequence. The detailed technical phase plan remains in `IMPLEMENTATION_PLAN.md` and backend-specific sequencing remains in `BACKEND.md`.

## 22. Definition of Done

HELIOS-X is ready for its intended MVP demonstration only when all applicable criteria below are met.

### Explorer

- [ ] Public users can access the Explorer without creating an investigation.
- [ ] RA/Dec and FOV are validated and normalized.
- [ ] Archive discovery returns actual observation coverage.
- [ ] Actual observation times are available and correctly ordered.
- [ ] Actual SPHEREx images/cutouts are displayed, or fixture content is explicitly labelled during development.
- [ ] Timeline, play, pause, and frame stepping work on real returned frames.
- [ ] Frame metadata and available quality flags are inspectable.
- [ ] Zero/one/multiple frame cases and retrieval failures are represented honestly.
- [ ] All five NASA target scenarios have recorded outcomes.
- [ ] Public query and artifact access limits are enforced.

### Agentic investigation

- [ ] A bounded investigation can be started from a selected Explorer region or observation set.
- [ ] Capability and quality checks run before dependent methods.
- [ ] At least one deterministic scientific analysis passes a known-answer test.
- [ ] Relevant agent roles use validated typed contracts and approved tools.
- [ ] Results preserve evidence, provenance, uncertainty, quality outcomes, and versions.
- [ ] Candidate assessment is deterministic and explainable.
- [ ] Reports are generated from persisted validated records.
- [ ] Failures, retries, cancellation, and historical results behave according to the contracts.

### Integrated platform

- [ ] Explorer and investigation layers share consistent source and observation semantics.
- [ ] Explorer works without agents; investigation works without trusting browser-supplied measurements.
- [ ] Investigation results link back to the exact Explorer query, observations, and image artifacts.
- [ ] API, database, frontend, backend, data-source, security, tests, and documentation contracts agree.
- [ ] Features are labelled as implemented, fixture-only, live-verified, planned, unavailable, or not supported accurately.
- [ ] No unsupported discovery claim is presented.

---

**End of Product Requirements Document.**
