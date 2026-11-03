# HELIOS-X — Investigation Workflow

> **Version:** 2.0  
> **Updated:** 2026-11-02  
> **Purpose:** Investigation execution workflow aligned with the integrated SPHEREx Sky Image Explorer and Agentic Scientific Investigation Engine.

## 1. Overview

The HELIOS-X investigation workflow converts a user request into a bounded, reproducible scientific investigation. It validates the request, checks data capabilities, retrieves observations, executes supported analyses, evaluates evidence, prioritizes candidates when appropriate, and generates a provenance-linked report for human review.

The workflow is controlled by a **Workflow Orchestrator**, which is a system component rather than an AI agent. Specialized AI agents perform planning and evidence interpretation, while deterministic scientific functions perform measurements, calculations, and statistical tests.

### 1.1 High-Level Workflow

HELIOS-X has two integrated but operationally separate paths. The Public Explorer is independently usable; the investigation workflow begins only from a direct investigation request or an explicit user handoff.

**Path A — Public SPHEREx Sky Image Explorer (independent):**

```text
Visitor enters RA / Dec + FOV
        ↓
Coordinate and query validation
        ↓
SPHEREx archive metadata / observation discovery
        ↓
Versioned frame manifest using actual acquisition timestamps
        ↓
Selected image cutouts / display derivatives
        ↓
Time-ordered frame viewer (play, pause, step, timeline, optional blink)

No investigation, candidate, or agent run is created by browsing alone.
```

**Path B — Agentic Scientific Investigation:**

```text
Direct Investigation Request
OR explicit "Investigate" action from the Explorer
        ↓
Request Intake and Validation
        ↓
Investigation Planner Agent
        ↓
Workflow Orchestrator Validates the Plan
        ↓
Data Capability Gate
        ↓
Data Retrieval and Metadata Collection
        ↓
Data Quality and Compatibility Checks
        ↓
Source Identification and Change Detection
        ↓
Conditional Scientific Analyses
   ├── Motion Analysis
   ├── Photometry and Variability
   ├── Spectral Analysis
   └── Other validated image/statistical operations
        ↓
Catalogue Cross-Matching
        ↓
Validation and Alternative-Hypothesis Evaluation
        ↓
Applicable Evidence Gates
        ↓
Candidate Assessment, If Applicable
        ↓
Provenance-Linked Report Generation
        ↓
Human Review and Recorded Decision
```

The Explorer may display image sequences even when scientific analysis is unsupported. A visual frame sequence or blink comparison is not itself a motion/variability finding. The Explorer must remain usable when the Agentic Engine is unavailable or the user has not started an investigation.

The investigation workflow is conditional rather than strictly linear. Tasks may execute independently when their dependencies permit it. Analyses that are unsupported, unnecessary, or blocked by missing prerequisites must not be executed as if they were valid. An Explorer handoff is a request to investigate, not proof that the selected observations are compatible or scientifically sufficient.

### 1.2 Governing Rules

- Every investigation must have a defined objective and scope.

- Every scientific analysis must pass its required capability and input checks.

- Measurements and statistical calculations must use versioned computational functions.

- AI agents must not invent observations, numerical results, or missing evidence.

- Every result must preserve its input references, method, status, and relevant uncertainty.

- Mandatory evidence gates must be enforced by the Workflow Orchestrator.

- Candidate prioritization must follow a documented policy rather than an opaque AI judgment.

- Automated prioritization must never be represented as confirmation of Planet X.
- Public Explorer browsing, searching, playback, or blinking must not implicitly create an investigation, candidate, or agent execution.
- An Explorer-to-investigation handoff must preserve a reproducible snapshot and revalidate every selected reference before analysis.
- Actual observation/acquisition time controls temporal ordering; retrieval, processing, release, filename, and display-rendering times are not substitutes.
- Spectral channels, detector arrays, broad bands, and observation epochs are separate data dimensions.
- A successful empty archive query, a failed query, partial retrieval, unsupported method, and insufficient data must remain distinguishable.

### 1.3 Agent Roles and System-Component Boundaries

The workflow uses the following eight logical AI-agent roles defined in `AGENT_SPECIFICATIONS.md`:

1. **Investigation Planner Agent** — proposes the bounded scientific plan.
2. **Data Retrieval Agent** — coordinates retrieval through approved data-source adapters and interprets retrieval outcomes.
3. **Data Quality Agent** — interprets deterministic schema, metadata, compatibility, and quality-check results.
4. **Change Detection Agent** — interprets validated detection and comparison outputs.
5. **Motion Analysis Agent** — interprets deterministic position/motion calculations.
6. **Spectral & Photometry Agent** — interprets validated photometric, variability, and spectral results.
7. **Catalogue Matching Agent** — interprets catalogue association results and ambiguity.
8. **Validation & Hypothesis Agent** — evaluates evidence, alternatives, contradictions, and remaining uncertainty.

The following are system components/services, not additional AI agents: **Workflow Orchestrator**, **Scientific Engine**, **Data-Source Adapters**, **Candidate Assessment Service**, and **Scientific Report Builder**. The Orchestrator owns control flow and mandatory gates. Adapters perform actual retrieval. The Scientific Engine performs validated calculations. Candidate Assessment applies a deterministic, versioned policy. The Scientific Report Builder composes reports from persisted results and provenance; optional language generation does not make it a ninth agent.

The eight roles are logical responsibilities and do not require eight separate LLM instances or microservices in the MVP. Their input/output contracts and failure behavior remain governed by `AGENT_SPECIFICATIONS.md`.

## 2. Step 1 — Request Intake

The system captures the information required to define the investigation:

- Investigation goal and scientific question.

- Target coordinates, object identifier, or sky region.

- Requested observation period, if applicable.

- Desired mission, archive, data source, or product, if known.

- Requested analysis types.

- Output format and level of detail.

- User-specified constraints.

- Any relevant scientific assumptions provided by the user.
- Request origin: direct investigation request or explicit Explorer handoff.
- Explorer query/snapshot reference, when applicable.
- Selected stable observation, product, frame, and artifact references; do not accept browser-supplied measurements as scientific evidence.
- Selected spectral channel/wavelength and detector-array filter as separate dimensions, when applicable.
- Archive/product release and processing constraints, if the user selected them.

### 2.1 Request Validation

Validate:

- Coordinate values, coordinate format, and reference frame.

- Observation time range and time conventions.

- Units and required parameters.

- Target identifier or sky-region definition.

- Compatibility between the requested objective and available workflow types.

If a critical parameter is missing or ambiguous, request clarification. A documented default may be used only when it is scientifically appropriate, clearly recorded, and does not materially change the user's intended question.

The system must not silently invent target coordinates, observation times, or scientific constraints.

For a named target, resolve the object through a documented authoritative resolver/catalogue where available. Preserve the original name, resolved identifier, component identity, coordinate frame, reference epoch, resolver/source, and uncertainty. Do not hard-code guessed coordinates. For Epsilon Indi B, explicitly disambiguate the intended system/component(s).

### 2.2 Investigation Record

Create an investigation record containing a unique identifier, the validated request, selected workflow type, initial status, creation timestamp, and applicable configuration version.

The original request must remain available for provenance even if the investigation plan is subsequently revised.

### 2.3 Explorer-to-Investigation Snapshot

When the user explicitly starts an investigation from the Explorer, create an immutable, versioned snapshot of the selected context. Preserve, where available:

- Original coordinate input and normalized coordinates.
- Coordinate frame, reference epoch, and target-resolution provenance.
- FOV geometry, units, region definition, and query filters.
- Source archive/interface, query parameters, query timestamp, query outcome, and returned-record count.
- Stable observation, product, frame, and artifact references.
- Actual acquisition timestamps with their time scale/format; never replace them with retrieval or processing time.
- Product family, data release, processing version, channel/wavelength, and detector-array identity as distinct fields.
- Quality/retrieval warnings, missing frames, and known capability limitations.

The handoff must send references and validated parameters, not large image arrays or client-invented measurements. The Investigation Service and Workflow Orchestrator must revalidate the snapshot, access permissions, current product references, data capability, compatibility, temporal coverage, and quality prerequisites. A frame selected for display is not automatically approved for scientific comparison. If the archive/release has changed since the snapshot, preserve the original snapshot and record any refreshed products as new evidence rather than silently replacing the original context.

Direct investigations by target name, coordinates, or region remain supported; they do not require the user to start from the Explorer.

## 3. Step 2 — Investigation Planning

The Investigation Planner Agent converts the validated request into a structured investigation plan.

The plan must specify:

- Scientific objective and scope.

- Required data products and metadata.

- Proposed analysis tasks.

- Task dependencies and execution order.

- Required quality checks and evidence gates.

- Expected outputs and result schemas.

- Known limitations and unresolved prerequisites.

- Conditions that would require clarification, additional data, or termination.

The planner may recommend analyses, but it must not claim that a dataset supports an analysis until the capability checks establish that support.

### 3.1 Plan Validation

The Workflow Orchestrator validates the proposed plan before execution.

It must verify that:

1. The plan addresses the requested objective.

2. Required inputs and dependencies are identified.

3. Proposed tasks use supported and available methods.

4. Mandatory validation gates are defined.

5. Unsupported or unnecessary analyses are excluded or explicitly marked conditional.

6. Resource limits and termination conditions are specified.

The orchestrator may reject the plan, request replanning, or block unsupported tasks.

**Responsibility boundary:** The Investigation Planner Agent proposes the plan. The Workflow Orchestrator controls execution, dependencies, retries, cancellation, status transitions, and mandatory gates.

## 4. Step 3 — Data Capability Check

Before retrieving or processing data, check whether the selected source and product support the requested investigation.

Evaluate:

- Whether the data product supports the intended measurement.

- Whether suitable observations and observation times exist.

- Whether required metadata and uncertainty information are available.

- Whether temporal, spatial, and spectral coverage are adequate.

- Whether the proposed method is implemented and validated for the selected data type.

- Whether documented product limitations affect the intended conclusion.
- Whether the selected release/product family is currently available and documented, and whether a newer release changes compatibility or interpretation.
- Whether actual observation epochs—not channel indices or retrieval timestamps—provide sufficient temporal coverage.
- Whether the selected channel/wavelength, detector-array identity, calibration, WCS, PSF, masks, variance, and uncertainty metadata support the intended operation.
- Whether any auxiliary catalogue or ephemeris service is actually available and appropriate for the requested check.

For example, do not promise source-motion analysis when the selected product lacks suitable repeated observations, valid observation times, or the astrometric information required by the method.

### 4.1 Capability Outcomes

Use the following outcomes consistently with the system architecture:

- `SUPPORTED`: Required capabilities and inputs are available.

- `NOT_SUPPORTED`: The requested analysis is unsupported by the selected product or implemented methods.

- `INSUFFICIENT_DATA`: Required observations, metadata, or coverage are inadequate.

- `BLOCKED`: A prerequisite or mandatory gate prevents execution.

An unknown capability must trigger a documentation or validation check. It must not be assumed supported.

The capability gate applies to scientific investigation tasks. It must not prevent the Public Explorer from displaying otherwise valid image products for visual inspection when motion, photometry, variability, subtraction, or another scientific method is unsupported. Such viewing must be clearly labelled as visual exploration, with limitations shown.

If a requested analysis is unsupported, the system may suggest an alternative only when that alternative is documented and suitable for the scientific objective. The substitution must be disclosed and must not silently change the meaning of the investigation.

## 5. Step 4 — Data Retrieval

Use the relevant data-source adapter to retrieve observations, metadata, and associated quality information.

The Data Retrieval Agent coordinates the request, prepares validated parameters, calls the approved adapter, interprets retrieval warnings and failures, and passes record references and provenance to downstream tasks. The adapter performs the actual archive/data retrieval; the agent must not fabricate records or mark retrieved products scientifically suitable.

Store:

- Query parameters and query timestamp.

- Data source, archive, and product identifiers.

- Product release or version.

- Observation identifiers and observation times.

- File references and checksums where feasible.

- Retrieval status and returned-record count.

- Warnings, missing records, and errors.

- Relevant documentation references.

### 5.1 Retrieval Validation

A successful HTTP response does not prove that the requested scientific data are valid or complete.

Verify that:

- The response corresponds to the intended query.

- Returned files or records are readable.

- Required fields and metadata are present.

- Product identifiers and versions are recorded.

- Partial retrievals and missing records are identified.

- Data integrity checks are performed where supported.

A failed query must remain distinguishable from a successful query that returns no matching records.

Retrieval retries must follow documented retry rules, with appropriate timeouts and limits. Retries must not create duplicate scientific results or conceal persistent failures.

### 5.2 Observation Discovery and Frame Manifest

For SPHEREx Explorer-derived investigations, use metadata-first discovery and build a versioned frame manifest from the products actually returned by the archive. Each manifest entry should include, where supplied:

- Stable HELIOS-X frame ID, archive observation ID, product ID, and artifact reference.
- Archive/source, mission/instrument, product family, release, and processing version.
- Actual acquisition/observation timestamp, time scale, and time representation.
- Normalized region center, coordinate frame/reference epoch, and actual FOV/cutout geometry.
- Broad wavelength band, spectral channel/wavelength, and detector-array ID as separate fields where the product exposes them.
- Image dimensions, pixel scale, units, WCS reference, calibration state, and available quality/contamination/mask/variance/PSF metadata.
- Retrieval timestamp/status, warnings, missing-record information, checksum/integrity reference where feasible, and original-product reference.

Order the temporal viewer by actual acquisition time, not by filename, product ID, array, channel, release, retrieval time, or processing time. Do not invent/interpolate missing frames or assume a uniform six-month cadence at a particular coordinate. Display gaps and partial retrievals honestly. Individual frame navigation must remain possible; a server-generated video is not required for the core Explorer.

Original scientific products must remain distinguishable from browser display derivatives such as thumbnails, rendered previews, or rescaled cutouts. Every derivative must retain its source-product and transformation lineage. A display derivative must not silently become the quantitative measurement input.

## 6. Step 5 — Data Validation and Compatibility

Validate retrieved data before scheduling dependent scientific analyses.

Deterministic validators produce schema, unit, coordinate, timestamp, quality-flag, and compatibility outcomes. The Data Quality Agent interprets these results, identifies analysis-specific restrictions, and reports unresolved issues. The Workflow Orchestrator, not the agent, enforces mandatory blocking rules.

Check:

- Schema and required fields.

- Units and measurement definitions.

- Coordinate systems and coordinate epochs.

- Observation timestamps and time conventions.

- Calibration status and quality flags.

- Missing values, masks, and upper limits.

- Uncertainty metadata.

- Product compatibility and relevant resolution differences.

- Data integrity and duplicate records.

- Suitability for the intended scientific method.

### 6.1 Quality-Check Outcomes

Each quality check must return one of these outcomes:

- `PASSED`: The check was performed and met its documented criteria.

- `FAILED`: The check was performed and did not meet its criteria.

- `INCONCLUSIVE`: The available result does not establish whether the criterion is met.

- `NOT_AVAILABLE`: Required information or capability is unavailable.

- `NOT_APPLICABLE`: The check does not apply to this analysis.

These outcomes describe individual checks, not task execution status.

If a mandatory check fails, block the affected analysis. If a check is unavailable or inconclusive, apply the documented policy for that check. Do not treat an unknown outcome as a pass.

Independent analyses may continue only when their own inputs and prerequisites remain valid.

### 6.2 Compatibility and Registration

When observations must be compared, confirm that their measurements are compatible. If image alignment is required, use a documented registration method and evaluate residuals against predefined criteria appropriate to the intended measurement precision.

If registration quality is inadequate, dependent position-change and motion analyses must be blocked or explicitly restricted.

### 6.3 SPHEREx-Specific Compatibility Rules

- Treat observation epoch as a time dimension; broad wavelength band and spectral channel/wavelength as spectral dimensions that must not be conflated; and detector-array identity as an instrument/product dimension. The six broad bands, 102 SPHEREx spectral channels, six detector arrays, and actual observation epochs are distinct dimensions. The 102 channels are not 102 epochs, and the six arrays are not six time samples or automatically interchangeable images.
- For temporal playback, retain distinct observations and order them using their actual acquisition timestamps. When comparing images quantitatively, group/filter frames to compatible broad band and channel/wavelength, product family, release/processing state, calibration, units, geometry, and quality state.
- Preserve detector-array identity when it is supplied. Any array combination is optional and may proceed only after product documentation and a validated method establish compatibility in calibration, units, WCS/alignment, PSF, background, masks/flags, variance, and weighting.
- Do not directly compare different releases or processing versions unless a validated compatibility method explicitly permits it. Record the chosen release and its basis.
- A preview, rescaled cutout, or colour-mapped display image may support visual inspection but is not automatically suitable for astrometry, photometry, or image subtraction.

## 7. Step 6 — Source Identification and Change Detection

Use supported source-detection methods or source records supplied by the selected data product.

The Change Detection Agent interprets validated detection and comparison results. Actual source detection, measurement, and change calculations must be performed by deterministic scientific functions.

For each relevant source and observation, record:

- Source and observation identifiers.

- Measured values and units.

- Measurement uncertainties.

- Method and implementation version.

- Detection thresholds and decision criteria.

- Quality-check outcomes.

- Rejected, duplicate, or ambiguous associations.

- Supporting evidence references.

Compare observations only when their measurement definitions and processing are compatible, or when a validated transformation is available.

A difference between observations must not automatically be interpreted as a real astrophysical change. Consider measurement uncertainty, calibration, registration, source confusion, and other relevant systematic effects.

If no suitable source records or observations are available, return an explicit status rather than manufacturing detections.

### 7.1 Visual Change Versus Scientific Change

Blinking, scrubbing, side-by-side viewing, or a visible pixel displacement is an Explorer aid only. It does not by itself establish source motion, variability, or a new object. Before assigning detections across epochs, use a documented source-association method and consider coordinate frame/epoch, proper motion, parallax where relevant, registration residuals, source blending, PSF changes, background, calibration, masks, and measurement uncertainty. A source that appears at a different pixel location may reflect WCS/registration or image-processing effects rather than physical motion.

The Change Detection Agent interprets outputs from deterministic detection, registration, comparison, and measurement functions. It must not promote visual differences or unvalidated client-side image analysis into scientific evidence.

## 8. Step 7 — Specialized Scientific Analyses

Run only the analyses required by the scientific question and supported by available data.

The Workflow Orchestrator schedules each analysis after verifying its dependencies and mandatory prerequisites.

### 8.1 Motion Analysis

Run when suitable positions, observation times, coordinate information, and uncertainty estimates are available.

The Motion Analysis Agent interprets the outputs of deterministic position and motion calculations.

Record angular displacement, elapsed time, estimated motion, uncertainty, method, quality checks, and limitations.

Do not present a preliminary motion estimate as a reliable orbit.

### 8.2 Photometry and Variability

Run when compatible flux measurements, relevant observation times, and suitable uncertainty information are available.

The Spectral & Photometry Agent interprets validated photometric results. Deterministic functions perform flux measurements, uncertainty propagation, SNR calculations, and applicable variability tests.

Record the measurement definitions, units, time sampling, statistical method, relevant quality checks, and limitations.

### 8.3 Spectral Analysis

Run when valid spectral measurements and required channel metadata are available.

The Spectral & Photometry Agent interprets the outputs of deterministic spectral calculations.

Record channel definitions, units, calibration, missing or invalid channels, uncertainty treatment, model assumptions, and limitations.

Do not infer physical properties such as temperature or composition unless the data and model support the inference.

### 8.4 Image Comparison, Difference, and Stacking

Visual blink/toggle comparison may be offered when at least two frames are available, but it must be labelled as visual comparison. A quantitative difference/subtraction product may run only when the input frames are compatible and the method has passed its validation gate. Depending on the product and operation, prerequisites include the same compatible channel/wavelength, product/calibration state, valid WCS, astrometric registration, unit/photometric-scale compatibility, background treatment, PSF matching, masks/contamination flags, variance/uncertainty handling, and documented subtraction diagnostics. Preserve input frame IDs, all transformation parameters, software/method version, residuals, and the derived artifact reference.

Image addition or stacking is a separate derived operation, not a replacement for original epoch frames. Combining different epochs may smear moving sources or suppress variability. Do not use a cross-epoch coadd as the sole input to a moving-object or variable-source investigation. Preserve the original time sequence and report the exact epochs included in every stack.

### 8.5 Optional Detector-Array Combination

Combining detector-array products is optional, not a default prerequisite for Explorer playback or scientific investigation. Keep per-array source products and provenance. Combine only when official product semantics and a validated method confirm compatibility across the relevant channel/wavelength, observation grouping, release/processing version, calibration, units, WCS, PSF, background, masks/flags, variance, exposure, and weighting. Never infer that all six detector arrays can be summed into a single scientifically valid image. Record the method, input array IDs, weighting, uncertainty propagation, and validation diagnostics. If prerequisites are not met, retain separate array products and report the combination as unavailable or unsupported.

### 8.6 Other Physical or Statistical Models

Any additional analysis must specify its required inputs, assumptions, implementation, validation criteria, and expected outputs before execution.

### 8.7 Analysis Task Results

Every analysis task must return:

- Task and investigation identifiers.

- Execution status.

- Input evidence identifiers.

- Result reference, where available.

- Method and software version.

- Configuration version where relevant.

- Warnings and errors.

- Output units and uncertainties where applicable.

- Start and completion timestamps.

Missing inputs must produce an explicit `INSUFFICIENT_DATA` or `BLOCKED` outcome, depending on the cause. Unsupported methods must return `NOT_SUPPORTED`.

A task must not be marked successful merely because it returned a response or produced a file.

## 9. Step 8 — Catalogue Cross-Matching

Query the configured catalogues using documented matching criteria.

The Catalogue Matching Agent coordinates supported catalogue queries and interprets match ambiguity. The underlying adapters and deterministic matching functions remain responsible for retrieval and numerical association calculations.

Record:

- Catalogue name and release/version.

- Query timestamp and parameters.

- Target position, coordinate epoch, and uncertainty.

- Matching method and search radius.

- Angular separation for each returned match.

- Relevant source attributes.

- Multiple-match ambiguity.

- Query success or failure status.

- Evidence references and known catalogue limitations.

Where supported, the matching method should account for positional uncertainty and local source density.

Interpretation rules:

- A match supports a possible association; it does not automatically prove identity.

- A successful query with no qualifying result means no match was found under the selected criteria.

- A failed or incomplete query does not establish that the source is unmatched.

- Multiple plausible associations must remain ambiguous unless sufficient evidence resolves them.

- Catalogue incompleteness must be considered when interpreting missing associations.

Catalogue cross-matching is an evidence source, not a standalone discovery test.

### 9.1 Auxiliary Catalogues and Context Layers

Gaia and WISE/NEOWISE are auxiliary external sources, not SPHEREx data feeds. Keep their catalogue release, observation/coordinate epoch, units, and provenance separate from SPHEREx products. A visual overlay and a formal catalogue cross-match are different operations; a positional overlay alone does not prove source identity. Aladin Lite may provide contextual sky imagery and catalogue/footprint overlays, but it does not replace the custom SPHEREx time-sequence viewer. Where MOST/JPL Horizons ephemerides are used, treat them as predicted positions for known Solar System objects under their stated assumptions; they are not a blind detector for unknown objects or Planet X.

## 10. Step 9 — Validation and Alternative Explanations

The Validation & Hypothesis Agent evaluates the available evidence against plausible alternative explanations.

Depending on the investigation, alternatives may include:

- Noise or low-significance detection.

- Calibration or image-registration error.

- Source blending or incorrect source association.

- Known moving objects or catalogue incompleteness.

- Instrumental artifacts or contamination.

- Duplicate detections.

- Insufficient temporal or spectral coverage.

- Inappropriate model assumptions.

- Other scientifically plausible explanations.

For each relevant alternative, record:

- Supporting evidence.

- Contradictory evidence.

- Tests performed and results.

- Tests unavailable or not performed.

- Remaining uncertainty.

- Recommended follow-up.

Distinguish between an alternative that has been evaluated and rejected under documented criteria, one that remains plausible, and one that could not be evaluated.

The agent may recommend additional tests but must not invent test results or override failed quality checks.

## 11. Step 10 — Evidence Gates

Use versioned, explicit evidence gates rather than an opaque AI judgment.

Each gate definition must specify:

- Gate identifier and policy version.

- Scientific question or claim to which it applies.

- Required inputs and evidence.

- Acceptance criteria.

- Whether the gate is mandatory.

- Applicable and non-applicable conditions.

- Handling of failed, unavailable, and inconclusive results.

- Downstream tasks or decisions affected by the outcome.

### 11.1 Initial Conceptual Gate Set

**G1 — Data Validity**

Required data, metadata, and product compatibility satisfy the documented criteria for the intended analysis.

**G2 — Measurement Quality**

Measurements and uncertainty estimates meet the documented requirements for the intended claim.

**G3 — Persistence or Repeatability**

The signal is supported by sufficient independent observations for the specific scientific question. Independence must be justified; repeated processing of the same observation does not automatically provide independent evidence.

**G4 — Motion or Change Evidence**

When the claim concerns motion or change, the relevant quantity is supported by an appropriate measurement method and statistical criterion.

**G5 — Known-Source and Alternative Checks**

Relevant catalogue associations and scientifically plausible alternative explanations have been evaluated to the extent required by the investigation.

**G6 — Independent Review Readiness**

The evidence, methods, uncertainties, quality checks, and limitations are sufficiently documented for human assessment.

These gates are a starting policy, not universal criteria for every investigation. For example, a spectral investigation may not require a motion gate.

### 11.2 Gate Outcomes

Use:

- `PASSED`: The applicable criteria were met.

- `FAILED`: The applicable criteria were not met.

- `UNKNOWN`: Available evidence is insufficient to determine the outcome.

- `NOT_APPLICABLE`: The gate does not apply to the scientific question.

A gate must not pass merely because its check was skipped or its required evidence is unavailable.

The Workflow Orchestrator must enforce mandatory gates. A failed mandatory gate must block the decisions or downstream analyses that depend on it. An unknown mandatory gate must not be treated as passed.

A non-mandatory gate may permit limited progress when the documented policy allows it, but its unresolved outcome must remain visible in the final report.

## 12. Step 11 — Candidate Prioritization

Candidate Assessment uses a versioned, deterministic policy to prioritize candidates for further investigation or review.

The policy must:

- Define each assessment criterion.

- Separate mandatory requirements from ranking preferences.

- Preserve missing and unknown values.

- Consider measurement quality and uncertainty.

- Account for relevant contradictory evidence.

- Avoid double-counting correlated evidence.

- Document score calculation, thresholds, and policy version.

- Explain how unavailable checks affect the result.

- Avoid presenting an uncalibrated score as a probability.

The candidate score must be derived from recorded evidence and documented rules. An AI agent may explain the result or recommend next steps but must not silently alter the score or override mandatory gates.

### 12.1 Candidate Outcomes

Use the following candidate-level statuses:

- `NEEDS_MORE_DATA`: Additional data or evidence are required.

- `NOT_PRIORITIZED`: The candidate does not meet the current prioritization criteria.

- `PRIORITIZED_FOR_REVIEW`: The candidate meets the documented criteria for further review.

- `BLOCKED_BY_QUALITY_ISSUE`: A relevant quality problem prevents assessment under the current policy.

These are candidate statuses, not task execution statuses.

Candidate prioritization does not confirm a new astronomical object or establish the identity of Planet X. A priority score must not be interpreted as a discovery probability unless it comes from a validated probabilistic model.

## 13. Step 12 — Report Generation

The Scientific Report Builder (a system component, not a ninth AI agent) builds the report from persisted evidence and validated computational outputs, not from agent memory alone.

The report must include:

1. Investigation question and scope.

2. Data sources, product versions, and retrieval status.

3. Methods, software versions, and relevant configurations.

4. Measurements, units, and uncertainties.

5. Catalogue cross-match results and ambiguity.

6. Data-quality and compatibility checks.

7. Statistical tests and relevant assumptions.

8. Alternative hypotheses and unresolved contradictions.

9. Evidence-gate outcomes and rationale.

10. Candidate status and prioritization rationale, where applicable.

11. Missing evidence, limitations, and unavailable checks.

12. Recommended follow-up actions.

13. Provenance references for quantitative claims.

14. Human review status.

### 13.1 Claim Traceability

Each quantitative claim must link to the relevant computational result or evidence record.

The report must distinguish among:

- **Observation:** What the data product records.

- **Calculation:** What a documented method derives.

- **Interpretation:** What the evidence supports under stated assumptions.

- **Hypothesis:** A possible explanation that remains subject to testing.

- **Limitation:** What the available evidence cannot establish.

- **Recommendation:** A proposed next investigative step.

Agent-generated summaries must not be presented as direct observations or validated measurements.

Report generation may proceed with limitations when the policy permits, but unresolved prerequisites and missing evidence must remain explicit.

A report marked ready for review indicates that the report is available for human assessment; it does not indicate scientific acceptance or discovery confirmation.

For an Explorer-derived investigation, the report must link back to the saved Explorer query/snapshot and selected frames. Quantitative image-operation claims must reference the exact input frame IDs and derived-product lineage. The report must identify whether an image was an original product or a display derivative. The retrieval timestamp, file timestamp, release timestamp, or rendering timestamp must never be described as the observation time.

## 14. Step 13 — Human Review

A human reviewer may:

- Accept the report as a valid investigation summary.

- Reject or revise a candidate interpretation with a documented rationale.

- Request a new analysis.

- Request additional observations.

- Mark the investigation inconclusive.

- Add a reasoned annotation.

- Request clarification of methods, assumptions, or evidence.

Record the review decision, timestamp, reviewer identifier where permitted, rationale, and evidence or report version reviewed.

Human review must not silently overwrite the underlying scientific measurements. Any requested change to an analysis must create a new run or a versioned annotation.

A report may be accepted as a valid summary without confirming its scientific hypothesis. The review outcome must make this distinction clear.

## 15. Re-analysis and New Evidence

New observations, corrected metadata, updated catalogues, changed methods, or revised configurations may require a new analysis run.

The Workflow Orchestrator must:

1. Identify which inputs or assumptions changed.

2. Determine affected tasks from the dependency graph.

3. Re-run only the affected tasks and their necessary downstream dependencies when safe to do so.

4. Revalidate mandatory gates affected by the changes.

5. Preserve previous results and their provenance.

6. Create new versioned results rather than overwrite historical outputs.

7. Record why candidate statuses or priorities changed.

Each run must retain the relevant input identifiers, data versions, software and method versions, configuration, parent evidence IDs, and execution outcomes.

Unchanged results may be reused only when their dependencies and reproducibility conditions remain valid. A result must not be reused merely because its task name matches.

If a method, threshold, or candidate-ranking policy changes, the system must record the new version and identify whether existing investigations require re-evaluation.

## 16. Task Statuses, Dependencies, and Failure Handling

Task execution statuses must follow the system architecture's canonical enum:

- `QUEUED`

- `RUNNING`

- `SUCCEEDED`

- `FAILED`

- `RETRYING`

- `CANCELLED`

- `BLOCKED`

- `NOT_SUPPORTED`

- `INSUFFICIENT_DATA`

Task statuses must remain separate from quality-check outcomes, evidence-gate outcomes, and candidate statuses.

The Workflow Orchestrator must:

- Enforce task dependencies and mandatory gates.

- Prevent blocked tasks from executing as though prerequisites were satisfied.

- Retry only eligible failures under documented retry policies.

- Apply timeouts and resource limits.

- Support cancellation.

- Prevent duplicate execution from creating inconsistent evidence.

- Record errors and warnings.

- Preserve partial results where the task contract allows them.

- Explain why a task was skipped, blocked, or terminated.

An optional task failure must not automatically invalidate independent successful analyses. However, any conclusion that depends on the failed task must be restricted or marked unresolved.

## 17. End Conditions

An investigation may terminate when:

- A report is ready for human review.

- Required data are unavailable.

- A mandatory quality or evidence gate fails.

- Evidence remains insufficient.

- The requested analysis is unsupported.

- The user cancels the investigation.

- A documented resource or execution limit is reached.

- An unrecoverable workflow failure prevents further valid progress.

Every terminal outcome must include a clear status and explanation.

The system must distinguish a completed investigation with inconclusive evidence from an investigation that failed to execute. A lack of evidence must not be reported as evidence of absence.

Where partial results are available, they may be retained and reported only when their limitations and completion status are explicit.

## 18. Workflow Acceptance Criteria

The workflow is acceptable when:

- A validated user request produces a bounded, traceable investigation plan.

- The Workflow Orchestrator validates the plan and enforces task dependencies.

- Unsupported analyses are rejected before invalid execution.

- Data retrieval failures are distinguishable from valid empty query results.

- Mandatory quality checks cannot be silently skipped.

- Independent analyses can continue when unrelated tasks fail and their own prerequisites remain valid.

- Scientific calculations use documented, versioned computational functions.

- Task statuses, quality outcomes, evidence-gate outcomes, and candidate statuses remain distinct.

- Every result and quantitative report claim links to its supporting evidence.

- Candidate prioritization follows a versioned policy.

- Re-analysis preserves previous results and creates traceable new versions.

- Human review decisions and their rationale are recorded.

- Missing, inconclusive, or unavailable evidence is never silently treated as positive evidence.

- No automated workflow confirms Planet X or another new planet solely from candidate prioritization.

### 18.1 Public Explorer Independence

- A visitor can query a supported sky position and FOV and inspect actual returned SPHEREx observations without creating an account-bound investigation or starting any agent run.
- The frame sequence uses actual acquisition timestamps and keeps spectral channel/wavelength and detector-array identity separate from time.
- The viewer displays one-frame, multiple-frame, empty-result, partial-result, incompatible-product, and provider-failure cases truthfully; it does not synthesize missing observations.
- Frame metadata and quality limitations are inspectable, and failed frames are not silently skipped or reordered.
- If the Agentic Engine is unavailable or an analysis is unsupported, valid Explorer browsing remains available.

### 18.2 Explorer-to-Investigation Handoff

- Only an explicit user action creates an investigation from Explorer context.
- The handoff persists normalized and original coordinates, FOV, filters, selected stable observation/product/frame references, actual timestamps/time conventions, release/processing identity, channel/wavelength, detector array, source/query provenance, and warnings where available.
- The backend revalidates references, permissions, product availability, temporal coverage, compatibility, quality, and analysis prerequisites before executing scientific tasks.
- The handoff does not accept large client image payloads or client-supplied measurements as authoritative scientific evidence.
- An unavailable or unsupported analysis does not erase the Explorer result or invalidate independent valid analyses.

### 18.3 NASA Challenge Target Validation

The required target suite is:

| Target | Primary workflow behavior to validate |
|---|---|
| Barnard's Star | Resolve from an authoritative coordinate source; retain reference epoch/proper-motion handling; inspect actual archive coverage; run motion analysis only if astrometric and temporal prerequisites pass. |
| Epsilon Indi B | Verify target/component identity and whether the product resolution/blending supports separate-source interpretation; do not assume the pair is resolved. |
| WISE 0855 | Preserve the challenge-supplied input coordinates: RA `08h 55m 10.831s`, Dec `−07° 14′ 42.53″`; validate parsing, archive query, actual observation sequence, and motion-analysis capability. |
| RW Aurigae | Inspect actual observations and photometric suitability; run variability analysis only with compatible measurements, actual times, and suitable uncertainties. |
| R Draconis | Inspect actual observations and photometric suitability; run variability analysis only with compatible measurements, actual times, and suitable uncertainties. |

For targets whose coordinates are not supplied by the challenge, resolve them through an authoritative identifier/catalogue and retain source, release, alias/component identity, coordinate frame, epoch, and uncertainty. Do not invent or hard-code guessed coordinates. A target test passes when HELIOS-X correctly reports actual coverage, product validity, limitations, and method capability. A positive motion/variability detection is not required. A successful empty query, insufficient data, unsupported method, ambiguous component, and provider failure must each produce their correct distinct outcome.

### 18.4 Scientific Integrity and Provenance

- 102 spectral channels are never treated as 102 independent observation epochs.
- Detector-array products are not combined by default; any coaddition has validated compatibility and reproducible per-array provenance.
- Visual blinking is not reported as a measured motion/variability result.
- Image subtraction, stacking, photometry, and spectra use validated scientific inputs, methods, uncertainty handling, and stored derivation metadata.
- A catalogue no-match is not evidence of a new object or Planet X.
- Every result and quantitative report claim can be traced to the exact query, product, observation, frame, method, configuration, and version that produced it.

## 19. Relationship to Other HELIOS-X Documents

This document defines investigation execution logic. It must remain consistent with the integrated product requirements and the following contract owners:

- `PRD.md` — Product goals, user journeys, scope, Explorer independence, and acceptance requirements.
- `SYSTEM_ARCHITECTURE.md` — Component boundaries, execution ownership, Explorer/investigation flows, canonical status models, and data flow.
- `AGENT_SPECIFICATIONS.md` — The eight logical agent roles, input/output contracts, approved tools, and failure behavior.
- `SCIENTIFIC_METHODOLOGY.md` — Scientific assumptions, measurement rules, compatibility, uncertainty, evidence, and validation principles.
- `DATA_SOURCE_SPEC.md` — SPHEREx products/releases, archive adapters, frame manifests, target tests, auxiliary sources, and retrieval behavior.
- `BACKEND.md` — Backend service behavior, execution and persistence responsibilities, artifact delivery, and API-side enforcement.
- `API_SPECIFICATION.md` — Canonical request/response schemas, identifiers, and endpoint contracts.
- `DATABASE_SCHEMA.md` — Persistence entities, relationships, status fields, and provenance records.
- `TESTING_AND_ACCEPTANCE.md` — Test cases, validation datasets, tolerances, and implementation acceptance criteria.
- `FRONTEND.md` — Public Explorer interaction, explicit handoff UX, and user-facing behavior.

Responsibility boundary: this document describes when and in what dependency order investigation tasks execute. It does not redefine API schemas, database columns, agent tool permissions, source-specific retrieval protocols, or scientific algorithms owned by the documents above. Use the canonical contract from the owning document rather than introducing a competing enum, field, or method here.

Any conflict between these documents must be resolved explicitly rather than silently introducing a new scientific rule, workflow responsibility, or status value.


---

## Update Notes — Version 2.0

This update preserves the original investigation stages and conditional execution model. Changes align the workflow with the integrated product requirements and current companion specifications:

- Separates standalone Public Explorer browsing from explicit/direct investigation execution.
- Adds Explorer-to-investigation snapshot and revalidation requirements.
- Clarifies actual acquisition-time ordering, versioned frame manifests, and SPHEREx epoch/channel/array dimensions.
- Adds product-specific compatibility and image-operation gates for visual comparison, subtraction, stacking, and optional detector-array combination.
- Clarifies the eight-agent boundary and corrects report generation to the non-agent Scientific Report Builder service.
- Adds the five NASA target validation expectations without requiring positive detections.
- Aligns provenance, target tests, and document ownership with PRD, Architecture, Backend, Data Source Spec, and Scientific Methodology.

The workflow remains conditional: unavailable data, unsupported methods, failed checks, and inconclusive evidence must be reported honestly. No change authorizes an automated Planet X discovery claim.
