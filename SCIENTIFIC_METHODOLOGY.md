# HELIOS-X — Scientific Methodology

> **Document version:** 2.0  
> **Status:** Updated — aligned with the integrated SPHEREx Sky Image Explorer and Agentic Scientific Investigation Engine  
> **Last updated:** 2026-11-02  
> **Primary authority:** This document defines scientific-method behavior. Product scope is owned by `PRD.md`; data-source semantics by `DATA_SOURCE_SPEC.md`; architecture and service boundaries by `SYSTEM_ARCHITECTURE.md`; execution order by `INVESTIGATION_WORKFLOW.md`; agent responsibilities by `AGENT_SPECIFICATIONS.md`.

## 1. Purpose and Governing Principles

This document defines the scientific rules that HELIOS-X must follow when investigating astronomical observations. The system must use documented data products, explicit assumptions, reproducible computations, uncertainty-aware interpretation, and traceable evidence.

**Core principles:**

- AI agents may plan investigations, coordinate tasks, interpret validated results, evaluate alternative explanations, and recommend next steps.

- Scientific measurements, statistical tests, and numerical transformations must be performed by versioned computational functions.

- Every scientific claim must be traceable to supporting evidence, methods, assumptions, and uncertainty.

- Data limitations and failed checks must remain visible throughout the investigation.

- No candidate may be treated as a confirmed discovery solely because an automated system prioritizes it.

The scientific methodology defines required behavior. Specific algorithms, numerical thresholds, and acceptance limits must be selected, documented, and validated for the data product and scientific question before operational use.

### 1.1 Relationship to the Two Product Capabilities

HELIOS-X has two integrated but operationally distinct capabilities:

- **SPHEREx Sky Image Explorer:** lets a user query a sky region, inspect actual available SPHEREx image products, and browse the returned observations in time order. This is an exploratory viewing workflow and must work independently of an investigation.
- **Agentic Scientific Investigation Engine:** performs a bounded, explicitly requested scientific investigation using qualified inputs, deterministic scientific functions, validation gates, and traceable evidence.

Viewing, scrubbing, blinking, or comparing frames in the public Explorer MUST NOT by itself create an investigation, candidate, or agent execution. The Explorer may expose an explicit handoff action, but scientific analysis begins only after the user requests an investigation.

### 1.2 Explorer-to-Investigation Handoff

When a user explicitly starts an investigation from the Explorer, HELIOS-X must preserve a reproducible snapshot of the selected context, including the normalized coordinates, original coordinate input, coordinate frame/reference metadata, FOV, filters, selected frame/product references, observation identifiers and timestamps, data-source/release identity, and relevant warnings. The investigation must revalidate the snapshot against current capability and quality requirements; an Explorer display decision is not proof that the selected frames are scientifically compatible.

The Explorer remains usable when an investigation capability is unsupported or the user is not signed in. Failure or unavailability of the Agentic Engine must not invalidate otherwise available public image browsing.

## 2. Data Product Qualification

Before analysis, record:

- Mission, instrument, archive, and product name.

- Product documentation version or URL.

- Observation identifiers and observation times.

- Coordinate reference system and coordinate epoch, where applicable.

- Units, calibration status, and quality flags.

- Spatial and spectral resolution where relevant.

- Missing-data, masking, and uncertainty conventions.

- Product limitations and permitted use.

- Data retrieval timestamp and product version.

- Available measurement capabilities and known processing limitations.

Do not assume that a survey product supports source-motion analysis, precision astrometry, photometric variability analysis, or direct multi-epoch comparison. Confirm these capabilities from the product documentation and validate them with suitable test data.

For SPHEREx, distinguish spectral channels from observation epochs. The commonly described 102 infrared spectral channels are not 102 independent time samples. The actual usable channels, observation times, products, and uncertainties depend on the selected data product.

### 2.1 Data Capability Gate

Before scheduling a scientific analysis, the Workflow Orchestrator must verify that the required inputs, metadata, and supported computational methods are available.

The capability check must evaluate:

1. Whether the selected product supports the intended analysis.

2. Whether required fields and metadata are present.

3. Whether the observation count and temporal baseline are suitable.

4. Whether the relevant uncertainty and quality information is available.

5. Whether the proposed method has been implemented and validated for the input data type.

Use the following outcomes:

- `SUPPORTED`: Required capabilities and inputs are available.

- `NOT_SUPPORTED`: The selected data product or implementation does not support the requested analysis.

- `INSUFFICIENT_DATA`: Required observations, metadata, uncertainty information, or coverage are missing or inadequate.

- `BLOCKED`: A required prerequisite or quality gate has failed.

These outcomes must not be treated as interchangeable. A failed data query, for example, must not be interpreted as evidence that no source exists.

The system must not silently substitute a different method or fabricate missing inputs. Any fallback method must be documented, justified, and validated.

### 2.2 SPHEREx Dimensions and Time Semantics

For every SPHEREx analysis, keep the following dimensions separate:

- **Observation epoch/time:** the actual time at which a product's observation was acquired; use this for temporal ordering and time-domain analysis.
- **Spectral channel/wavelength:** one sample along the spectral dimension. The 102 SPHEREx spectral channels are not 102 independent observation epochs.
- **Broad wavelength band:** a product/instrument grouping that must not be confused with an individual channel or an observation time.
- **Detector-array identity:** an instrument/product dimension. Preserve the array identity where exposed; do not treat an array as an epoch, channel, or band.
- **Retrieval time:** when HELIOS-X queried or downloaded the data; it is not the observation time.
- **Release/processing version:** identifies the data product and processing lineage; it is not a new observation epoch.

The challenge's approximately six-month all-sky mapping description is not a guarantee that every coordinate has observations exactly six months apart, or even multiple usable observations in the currently available archive. Use actual regional coverage and acquisition timestamps. Preserve the timestamp representation and time scale; do not sort by filename, retrieval order, release date, or channel number. If acquisition-time metadata is missing or ambiguous, do not invent a precise chronology. Missing epochs are coverage gaps; do not interpolate or fabricate observations.

### 2.3 Source Product and Product-Release Qualification

The selected release's official documentation and FITS supplement govern the interpretation of product structure and metadata. Validate the actual product returned at runtime; do not assume that every release or product family has identical extensions, fields, calibration, uncertainty, or quality semantics. Record the release and processing version used for every result, and do not silently combine incompatible releases. A validated cross-release transformation may be used only when its method and limitations are documented.

Where a FITS product is available, validate its integrity, HDU/extension layout, dimensions, data type, units/scaling, WCS, observation-time metadata, wavelength metadata, masks/flags, variance, PSF, and other relevant planes before relying on them. Names and semantics such as `IMAGE`, `FLAGS`, `VARIANCE`, `ZODI`, `PSF`, or `WCS-WAVE` are product-documentation-dependent examples, not a universal schema. Never infer a missing plane's meaning or mark a missing quality/variance plane as passed.

### 2.4 Capability, Retrieval, and Quality Outcomes Are Different

Keep three kinds of status distinct:

- **Capability outcome:** `SUPPORTED`, `NOT_SUPPORTED`, `INSUFFICIENT_DATA`, or `BLOCKED` for the requested scientific method.
- **Retrieval/query outcome:** successful results (including a successful empty result), provider failure, timeout, malformed response, or other documented retrieval state.
- **Quality-check outcome:** `PASSED`, `FAILED`, `INCONCLUSIVE`, `NOT_AVAILABLE`, or `NOT_APPLICABLE` for an individual check.

A successful query returning zero records is not the same as an archive failure, and neither establishes that an astronomical source does not exist. An `INCONCLUSIVE` or `NOT_AVAILABLE` quality check is not a pass. Workflow task execution status is governed by `INVESTIGATION_WORKFLOW.md` and the common agent contract; it must not be overloaded to represent scientific quality.

## 3. Observation Compatibility and Alignment

Before comparing observations:

1. Confirm that the products measure compatible quantities.

2. Check coordinate systems, coordinate epochs, units, calibration, masks, and quality flags.

3. Confirm that observation timestamps and time conventions are valid.

4. If image comparison is required, use a documented WCS or astrometric registration procedure appropriate to the data.

5. Measure alignment residuals using suitable reference sources.

6. Record the registration method, software version, parameters, residuals, rejected reference points, and uncertainty estimates.

7. Verify that registration quality is adequate for the intended measurement precision before estimating apparent source motion.

### 3.1 Registration Acceptance Criteria

Registration acceptance limits must be defined before evaluating the result. They must be based on the intended scientific measurement, data resolution, reference-source quality, residual distribution, and required precision.

The validation must consider, where applicable:

- Residual offsets and their distribution.

- Spatially dependent systematic errors.

- Reference-source count and spatial coverage.

- Reference-source quality and possible misidentifications.

- Propagated registration uncertainty.

- Whether residuals satisfy the requirements of the downstream analysis.

A universal registration threshold must not be assumed appropriate for every dataset.

If registration fails the predefined criteria, motion and positional-change analyses that depend on it must be blocked or explicitly restricted. A visible shift must not automatically be interpreted as physical motion.

### 3.2 Compatibility for Temporal and Image Operations

Compatibility is operation-specific. Before a temporal comparison, verify the actual acquisition timestamps, sky region, coordinate frame/reference epoch, WCS, image geometry/pixel scale, product family, release/processing state, units/calibration, selected spectral channel or wavelength, detector-array identity where relevant, and applicable masks, variance, PSF, and quality information. A difference in one of these fields does not always make comparison impossible, but it requires a documented and validated transformation; otherwise the dependent operation must be blocked or marked unsupported/insufficient.

A basic Explorer sequence may show individually valid frames even when the frames are not suitable for scientific subtraction or astrometric measurement. Such a display must not imply that the images are registered or directly comparable. For registered/blink or quantitative comparison, preserve the source frames and report registration diagnostics and warnings. A visual blink is an exploratory aid, not a scientific measurement.

For image addition, subtraction, or stacking, document the objective and method. Apply appropriate background treatment, unit/scale normalization, masks, variance/uncertainty handling, and PSF matching where required. If measurements are correlated, account for covariance rather than assuming independent errors. Record all input frames and transformations. Do not create a difference or coadd merely because two frames are available.

## 4. Source Position and Motion

For two valid sky positions, calculate angular separation using spherical-coordinate methods when required by the geometry. A small-angle approximation may be used only when its assumptions are appropriate and documented.

For a sufficiently small angular displacement:

`Angular rate ≈ Angular displacement / Elapsed time`

The result must include the time baseline, units, direction or sign where meaningful, and the method used to calculate the displacement.

### 4.1 Uncertainty Propagation

Motion uncertainty must account for relevant sources of error, including:

- Positional measurement uncertainty.

- Image-registration uncertainty.

- Astrometric calibration uncertainty.

- Correlations between measurements.

- Time uncertainty where it materially affects the result.

- Other documented systematic effects.

When errors are independent and the assumptions of the selected method are satisfied, first-order uncertainty propagation may be appropriate. When errors are correlated, use a covariance-aware method or another justified uncertainty-propagation procedure.

The software must document the method, input uncertainties, relevant covariance assumptions, and resulting uncertainty. If a required uncertainty cannot be estimated reliably, report it as unavailable and restrict the interpretation accordingly.

### 4.2 Evidence Required for a Motion Claim

A motion claim requires:

- Valid source positions and observation times.

- Known coordinate frames and epochs.

- Acceptable registration and astrometric quality.

- Positional and relevant systematic uncertainties.

- A documented statistical test, fit criterion, or detection threshold.

- Checks for source confusion and known artifacts.

- Sufficient observational coverage for the stated conclusion.

Two detections alone may be insufficient to distinguish real motion from measurement error, source confusion, or systematic effects. The required number and timing of observations depend on the measurement precision and scientific question.

A preliminary angular-rate estimate must not be presented as a reliable orbit or definitive object classification.

### 4.3 Proper Motion, Parallax, and Apparent Displacement

A measured positional change across epochs is an apparent angular displacement until the relevant astrometric effects and systematic errors have been evaluated. Depending on the source and time baseline, interpretation may need to consider proper motion, parallax, observer/ephemeris effects, coordinate-epoch propagation, and source confusion in addition to measurement and registration uncertainty.

Do not apply a proper-motion correction merely because a target has a known name or is expected to move. Any coordinate propagation must use a documented astrometric source, reference epoch, proper-motion values, uncertainties/covariances where available, time convention, and a validated propagation method. Preserve the original target coordinates as well as any epoch-adjusted coordinates and their provenance.

A high apparent angular rate alone does not identify an object as a Solar System body, brown dwarf, planet, or Planet X. A preliminary angular-rate estimate is not an orbit. Orbital fitting requires suitable observations, temporal baseline, astrometric precision, geometry, and an explicitly validated model.

### 4.4 Source Association Across Epochs

A detection in one frame must not automatically be treated as the same source in another frame. Association must consider positional uncertainty, registration uncertainty, source morphology/quality, blending, crowding, duplicate detections, and the expected motion range for the hypothesis. Preserve rejected and ambiguous associations. If more than one association is plausible, report ambiguity rather than selecting the most convenient match.

## 5. Photometry, SNR, and Variability

Use the flux definition and uncertainty supplied by the data product or by a validated photometry method.

Where the flux uncertainty is valid and the noise model supports the calculation:

`SNR = Measured Signal / Estimated Signal Uncertainty`

Record the exact signal definition, uncertainty definition, units, background treatment, and calculation method. Do not compare SNR values from incompatible processing methods without justification.

### 5.1 Variability Analysis

For a variability investigation:

1. Check units, calibration, quality flags, upper limits, and observation timestamps.

2. Verify that measurements are comparable across observations.

3. Plot measurements with uncertainties against observation time.

4. Select a statistical test appropriate to the sampling pattern, noise model, and scientific question.

5. Record the test statistic, degrees of freedom where applicable, assumptions, and p-value or other calibrated output.

6. Account for relevant systematic errors and multiple testing.

7. Avoid interpreting statistical significance as proof of a particular physical cause.

The treatment of non-detections, upper limits, missing observations, and correlated measurements must be documented where applicable.

A light curve is only as reliable as its photometry, calibration, and time metadata.

### 5.2 Epoch Photometry and Display-versus-Measurement Separation

A displayed brightness difference is not a validated variability measurement. Variability analysis must use suitable calibrated flux measurements or a validated photometry procedure on scientific source data, with compatible measurement definitions, units, apertures/PSF treatment, backgrounds, quality masks, uncertainties, and actual observation times.

Do not calculate scientific flux or SNR from arbitrary browser previews, display-stretched pixels, JPEG/PNG derivatives, or per-frame auto-normalized images. A display derivative is for visualization unless a validated method explicitly establishes otherwise. Record the measurement extraction method and all per-epoch quality decisions.

### 5.3 Coaddition and Time-Domain Preservation

Detector-array combination or coaddition is optional and must never replace the original epoch frames. Combine detector-array products only when the official product semantics and validated method support compatible calibration, WCS/alignment, units, PSF, background, masks, variance, and weighting. Preserve per-array identity and the complete input lineage.

Combining distinct observation epochs may improve a static-source image under suitable conditions, but it can smear moving sources, suppress variability, or erase time-domain evidence. Do not coadd across epochs before preserving and exposing the original temporal information. Label any derived stack with its input epoch range and intended scientific use; evaluate whether stacking is inappropriate for the target hypothesis.

A visually brighter or smoother coadd is not by itself evidence of a higher scientifically meaningful SNR. SNR must be calculated from validated signal and uncertainty definitions with the applicable covariance and weighting assumptions.

## 6. Spectral Analysis

Before comparing spectral measurements:

- Confirm channel definitions, ordering, and wavelength or frequency metadata.

- Check units, calibration, masks, and uncertainty information.

- Identify missing, invalid, or flagged channels.

- Verify whether the values represent flux, intensity, or another physical quantity.

- Confirm whether the compared measurements are scientifically compatible.

- Apply only documented and validated conversions.

Spectral features may support comparisons between hypotheses, but they are not necessarily unique object identifiers.

Do not claim a temperature, composition, or physical classification unless the required data, calibration, physical model, and assumptions are documented and suitable for the intended inference.

For blackbody-based estimates, state that the model is an approximation and identify its limitations. Do not apply Wien's law or Planck's law to arbitrary channel values without verifying that the measurements and model assumptions are appropriate.

Where model fitting is performed, record the model definition, fitted parameters, parameter uncertainties, goodness-of-fit assessment, and relevant alternative models. A model fit alone does not establish that the physical interpretation is unique.

### 6.1 Channel, Wavelength, and Spectral-Time Separation

Treat SPHEREx spectral channels as wavelength samples, not a time sequence. A channel-selection control belongs to the spectral dimension, not the observation timeline. Preserve channel identifiers, wavelength/bandwidth metadata, ordering, calibration, units, flags, and uncertainty for every spectrum or channel-level result. Do not infer an epoch change from a change in channel.

If detector-array-specific spectral information is available, preserve array identity and use only documented array/channel combinations. On-demand spectrum generation is conditional on the selected product and extraction method supporting it; it is not guaranteed for every image or target.

### 6.2 Spectral Interpretation and Cross-Epoch Comparison

Before comparing spectra across observations, confirm wavelength grids/channel definitions, spectral response/calibration, units, extraction apertures or source association, masks, uncertainties, and compatible processing. Document any resampling or calibration transformation. A spectral feature may support a hypothesis but is not necessarily a unique object identifier.

Do not infer temperature, composition, or object class from arbitrary channel values. Blackbody or other physical-model fitting is permissible only when the measurements, calibration, model assumptions, parameter uncertainties, goodness-of-fit, and plausible alternative models are suitable and documented.

## 7. Catalogue Cross-Matching

Every catalogue cross-match must record:

- Catalogue name and release/version.

- Query timestamp and query parameters.

- Candidate position, coordinate epoch, and uncertainty.

- Matching method, metric, and search radius.

- Angular separation for each returned match.

- Relevant source attributes and their uncertainties, where available.

- Multiple-match ambiguity and missing fields.

- Query completion status and any errors.

Use positional uncertainty and local source density when selecting a matching method. A fixed search radius must be justified or explicitly identified as a configurable test value.

Where supported, estimate association reliability using a documented method that accounts for positional uncertainty and the likelihood of chance associations. Do not interpret a positional match as definitive identity without sufficient supporting evidence.

### 7.1 Interpretation Rules

- A catalogue match supports a possible association; it does not automatically prove identity.

- No returned match means that no match was found under the selected query and criteria.

- A failed or incomplete query is not an unmatched source.

- Multiple plausible matches must remain explicitly ambiguous unless additional evidence resolves them.

- Catalogue incompleteness must be considered when interpreting a missing association.

The report must distinguish between a successful query with no qualifying result, an ambiguous result, and a query failure.

### 7.2 Auxiliary Catalogues, Surveys, and Context Layers

Auxiliary sources must remain distinct from SPHEREx observations and from one another:

- **Gaia:** an ESA catalogue/astrometry source. Record the exact release, reference epoch, proper motion/parallax, uncertainties and covariance where available. Propagate coordinates only through a documented method. Gaia non-detection is not proof that a source is new; coverage and sensitivity limitations must be considered.
- **WISE/NEOWISE:** an independent NASA infrared survey with its own bands, products, releases, and observing epochs. Preserve its mission phase, product/release, time range, units, and quality metadata. Do not label its observations as SPHEREx epochs or combine its measurements with SPHEREx photometry without a validated cross-instrument method.
- **Aladin Lite:** may provide contextual sky navigation, standard survey imagery, footprints, or catalogue overlays. A displayed Aladin background or marker is not a SPHEREx frame or a scientific cross-match. The custom SPHEREx time-sequence viewer remains the source for SPHEREx temporal playback.
- **IRSA MOST and JPL Horizons:** may assist with ephemerides or checks for known Solar System objects with valid identifiers. They do not provide an orbit for hypothetical Planet X and are not blind unknown-object discovery systems.

A visual overlay and a scientific catalogue association are separate operations. A nearby marker is not proof of identity; cross-match rules in Section 7 must still be applied.

## 8. Quality and Artifact Checks

Depending on the data product and analysis, check:

- Signal-to-noise and quality flags.

- Background variation.

- Point-spread function (PSF) effects and source blending.

- Saturation, detector artifacts, and contamination.

- Image edges and masks.

- Image-registration residuals.

- Missing or inconsistent metadata.

- Duplicate detections and incorrect cross-observation associations.

- Calibration inconsistencies and known systematic effects.

- Suitability of the source-detection and measurement procedures.

Not every check is available or applicable to every data product.

Each check must have an explicit outcome, such as:

- `PASSED`: The check was performed and met its documented criteria.

- `FAILED`: The check was performed and did not meet its criteria.

- `INCONCLUSIVE`: The available result does not establish whether the criterion was met.

- `NOT_AVAILABLE`: The required information or capability was unavailable.

- `NOT_APPLICABLE`: The check does not apply to the selected analysis.

A check must not be marked `PASSED` merely because it was skipped. A failed mandatory check must trigger the documented blocking, restriction, or review policy.

### 8.1 Product-Specific Pixel and FITS Quality

When the selected product provides them, inspect the semantics and validity of image, flag/mask, variance, background, PSF, and wavelength/WCS planes. Validate each plane's dimensions, units, scaling, coordinate relationship, and release-specific meaning. Do not assume that a plane is present in every release or product family, and do not treat a missing plane as a passed quality check.

Where a modeled background component (for example, zodiacal-light information) is supplied, determine from the selected product documentation whether it is already applied to the science image or provided separately. Do not subtract or add it based on the extension name alone.

### 8.2 Preview, Cutout, and Scientific Source Integrity

Preserve the original archive product or a durable reference and identity for it. A cutout, resampled image, thumbnail, or browser preview is a derived artifact. Record its parent product, region, channel, dimensions, scaling/stretch, normalization, color mapping, masks, reprojection/resampling, software version, and checksum when computed.

Scientific measurement must use the authoritative source product or a documented lossless/validated intermediate. Display transformations must not be mistaken for physical changes in flux or source position. Any result based on a derived artifact must identify the derivation and establish that the artifact is suitable for that specific measurement.

## 9. Statistical Testing

Each statistical test must document:

- The scientific question and null hypothesis.

- The test statistic and implementation version.

- Required inputs and assumptions.

- The decision threshold or rule and its rationale.

- Treatment of uncertainty, covariance, and systematic effects.

- Multiple-testing considerations when many sources or hypotheses are evaluated.

- The result, uncertainty, and limitations.

- Whether the test was completed successfully.

Use chi-square only when the model and error assumptions justify it. For example, a goodness-of-fit statistic may be expressed as:

`χ² = Σ ((observed_i - model_i)² / uncertainty_i²)`

This expression is appropriate only when the uncertainties, model, and independence assumptions justify it. When observations are correlated, an appropriate covariance-aware formulation must be used.

The software must record the fitted model, included observations, degrees of freedom, uncertainty treatment, and any covariance handling. The exact method must be appropriate to the scientific question.

### 9.1 Multiple Testing

When screening many sources, channels, time series, or hypotheses, evaluate the risk of false positives caused by multiple testing.

The selected correction method must depend on the scientific objective, dependence structure, and statistical assumptions. A method such as False Discovery Rate (FDR) control may be appropriate for some screening tasks, but it must not be applied automatically without considering its assumptions.

For every multiple-testing procedure, record:

- The correction method and implementation version.

- The number and definition of tests included.

- The significance or control threshold.

- The assumptions and dependencies considered.

- The adjusted statistical outputs and decision rule.

The correction procedure must be defined consistently for comparable investigations. Any exploratory analysis performed without a suitable correction must be clearly labelled as exploratory.

Statistical significance is not equivalent to scientific importance, causal explanation, or confirmed discovery.

## 10. Alternative-Hypothesis Evaluation

The Validation & Hypothesis Agent must consider plausible alternative explanations appropriate to the available data. These may include:

- Noise or low-significance detection.

- Calibration or image-registration error.

- Source blending or incorrect source association.

- Known moving objects or catalogue incompleteness.

- Instrumental artifacts or contamination.

- Insufficient temporal coverage.

- Incorrect source detection or duplicate detections.

- A real astronomical source whose observed behavior does not support the proposed identity.

- Limitations of the selected physical or statistical model.

For each relevant alternative, record:

- Evidence supporting the alternative.

- Evidence against the alternative.

- Tests performed and their results.

- Tests that were unavailable or not performed.

- Assumptions and limitations.

- Remaining uncertainty.

- Recommended follow-up analysis, where appropriate.

The agent must distinguish between an alternative that has been tested and rejected under documented criteria, one that remains plausible, and one that could not be evaluated.

Do not force every candidate into a binary “real” or “false” decision when the evidence is inconclusive.

### 10.1 SPHEREx and Time-Domain Alternative Explanations

For SPHEREx image sequences and derived comparisons, evaluate applicable alternatives such as:

- WCS distortion, astrometric solution differences, or inadequate image registration;
- pixel-scale, reprojection, resampling, or display-stretch differences;
- PSF variation, source blending, crowding, or incorrect cross-epoch association;
- background gradients, zodiacal-light treatment, and background-subtraction differences;
- calibration, gain/response, unit, release, or processing-version differences;
- detector artifacts, bad/saturated pixels, cosmic-ray-like artifacts where relevant, masks, and contamination;
- changes in channel/wavelength or detector-array identity that were mistaken for temporal change;
- coaddition across epochs that smears motion or suppresses variability;
- auxiliary-catalogue epoch mismatch, incompleteness, or incorrect coordinate propagation.

Only alternatives relevant to the selected product and analysis need to be tested, but the report must state which were tested, unavailable, not applicable, or left unresolved. Do not present a skipped check as a rejected explanation.

## 11. Candidate Assessment and Planet X Policy

Planet X remains hypothetical. HELIOS-X must not automatically claim its discovery or identify an unmatched source as Planet X.

Acceptable wording includes:

- “Unmatched source candidate.”

- “Candidate with apparent motion.”

- “Prioritized for independent verification.”

- “Evidence is insufficient to establish identity.”

- “The available observations are consistent with multiple explanations.”

### 11.1 Candidate Prioritization

A candidate may be prioritized only under a documented assessment policy that considers relevant factors, including:

- Measurement quality and uncertainty.

- Completeness of required evidence.

- Reliability of source associations.

- Motion, photometric, or spectral evidence, where applicable.

- Alternative explanations and unresolved contradictions.

- Data limitations and the availability of independent verification.

- Whether mandatory quality and validation gates have passed.

The policy must define:

1. The criteria used for assessment.

2. Any weights, thresholds, or ranking rules.

3. How missing evidence is handled.

4. How contradictory results affect prioritization.

5. Which quality gates are mandatory.

6. How policy and configuration versions are recorded.

A candidate must not receive a higher priority merely because missing evidence is interpreted as positive evidence. Missing, unavailable, and failed checks must remain distinguishable.

Candidate scores must not be presented as discovery probabilities unless they are produced by a validated probabilistic model with documented assumptions and calibration.

Candidate prioritization is an investigative decision, not a scientific confirmation.

### 11.2 Independent Verification

High-priority candidates should be evaluated through independent evidence or follow-up observations appropriate to the hypothesis. Where possible, verification should use independent data, methods, or observations rather than merely repeating the same processing pipeline.

Any orbital analysis must be conditional on sufficient and suitable observations. It must document the observations used, model assumptions, fitted parameters, uncertainty, and limitations.

A preliminary motion estimate must not be presented as a reliable orbit. Automated results must not be reported as confirmation of Planet X or any other new planet without the required scientific evidence and independent expert assessment.

### 11.3 SPHEREx Candidate and Planet X Interpretation Boundary

An unmatched SPHEREx source is only unmatched under the recorded catalogue/query criteria. A catalogue no-match, including a Gaia or WISE no-match, is not evidence of a newly discovered object. The system must consider catalogue coverage, depth, epoch, coordinate propagation, source class, and query completeness before interpreting absence of a counterpart.

For a possible Solar System object, use known-object ephemeris services only for valid known identifiers or supported known-object checks. The absence of a match in such a service does not create an orbit or establish a Planet X candidate. Any candidate remains subject to alternative-hypothesis evaluation, independent observations, and qualified human review. The system must never expose an automated confirmed-planet or confirmed-Planet-X state.

## 12. Evidence Provenance and Reproducibility

Every scientific result must preserve, where applicable:

- Input dataset and record identifiers.

- Data source, retrieval timestamp, and product version.

- Input checksums or equivalent integrity identifiers where feasible.

- Code, dependency, and algorithm versions.

- Configuration, parameters, and thresholds.

- Random seeds for stochastic computations.

- Model and prompt versions for agent-generated summaries.

- Output values, units, uncertainties, warnings, and failure status.

- Parent evidence IDs and relationships to derived results.

- The method used to produce each quantitative result.

- Relevant validation and quality-check outcomes.

Re-running an analysis must create a new version of the result rather than silently overwrite historical evidence.

The system should preserve the relationship between observations, computational results, agent interpretations, candidate assessments, and report claims. An agent-generated claim must reference the evidence supporting it, and its interpretation must remain distinguishable from the underlying measurement.

Where exact reproducibility is not possible because of external data changes, stochastic methods, or environment differences, record the cause and expected limitations.

### 12.1 Explorer and Investigation Evidence Lineage

For Explorer retrieval and every investigation derived from it, preserve the exact query/region and filter context, source archive/interface, query and retrieval timestamps, query status and record count, product/observation/frame IDs, actual acquisition-time representation and time scale, release/processing version, channel/wavelength, detector-array identity, original product reference, derived cutout/preview lineage, and validation warnings.

For derived image operations, preserve all input-frame IDs, operation objective, registration/reprojection/background/PSF/mask/weighting parameters, software/method version, residuals and diagnostics, uncertainty treatment, and output artifact reference. The retrieval timestamp, release date, filename timestamp, and display-rendering timestamp must never be substituted for acquisition time.

## 13. Scientific Validation and Acceptance Criteria

HELIOS-X must satisfy the following criteria before a scientific workflow is treated as operationally validated.

### 13.1 Method and Measurement Validation

- Every quantitative result has a defined method and units.

- Uncertainty is reported or explicitly marked unavailable.

- Data compatibility and required capabilities are checked before analysis.

- Statistical tests document assumptions, inputs, and decision criteria.

- Failed, inconclusive, unavailable, or inapplicable checks are not counted as passed.

- Alternative explanations and unresolved contradictions are documented.

- Automated agents do not independently invent or alter scientific measurements.

- Planet X is never confirmed by an automated agent.

### 13.2 Software and Workflow Tests

Where applicable to the implemented analysis, validation tests must include:

1. **Synthetic-data recovery:** Test whether known source positions, shifts, or flux changes can be recovered within predefined tolerances.

2. **Uncertainty validation:** Check whether uncertainty propagation behaves as expected for controlled inputs and documented assumptions.

3. **Registration validation:** Test alignment procedures against known transformations or suitable reference data.

4. **Statistical validation:** Verify test statistics and decision rules against controlled examples or trusted reference implementations.

5. **Catalogue-query behavior:** Confirm that successful no-match results, ambiguous matches, and failed queries produce distinct outcomes.

6. **Capability-gate behavior:** Confirm that unsupported analyses return `NOT_SUPPORTED` and inadequate inputs return `INSUFFICIENT_DATA` where appropriate.

7. **Quality-gate enforcement:** Confirm that failure of a mandatory prerequisite prevents dependent analysis from being incorrectly marked successful.

8. **Reproducibility:** Confirm that the same input, software version, and configuration produce results consistent with the method's documented reproducibility requirements.

9. **Evidence traceability:** Confirm that quantitative outputs and report claims link to their supporting inputs and computational results.

10. **Regression testing:** Confirm that software changes do not silently alter validated behavior beyond documented tolerances.

Numerical tolerances, test datasets, and acceptance thresholds must be defined for each implemented method before its validation results are accepted.

### 13.3 Reporting Requirements

Every scientific report must distinguish among:

- **Observation:** What the selected data product records.

- **Calculation:** What a documented computational method derives from the observations.

- **Interpretation:** What the results support under stated assumptions.

- **Hypothesis:** A possible scientific explanation that remains subject to testing.

- **Limitation:** What the available data or methods cannot establish.

- **Recommendation:** What should be investigated or observed next.

The report must identify missing evidence, failed checks, unresolved alternatives, and relevant uncertainty. It must not present a candidate ranking, statistical significance, or agent-generated interpretation as a confirmed discovery.

### 13.4 SPHEREx Explorer and NASA Target Validation

The Public Sky Image Explorer must pass its own validation independently of the Agentic Scientific Investigation Engine. Explorer acceptance verifies coordinate/FOV handling, actual archive query behavior, product/cutout integrity, frame manifest correctness, timestamp ordering, honest coverage gaps, and truthful zero/one/multiple-frame states. It does not require a positive scientific detection.

The following NASA challenge targets form the required target-validation suite:

| Target | Scientific relevance | Required methodology checks |
|---|---|---|
| Barnard's Star | Nearby high-proper-motion star | Resolve its identifier and coordinates from an authoritative source; record reference epoch and proper-motion treatment; inspect actual SPHEREx coverage; run positional analysis only if its astrometric prerequisites pass. |
| Epsilon Indi B | Brown-dwarf pair/system | Resolve the intended component or system explicitly; inspect whether the selected SPHEREx product can resolve or separate the components; do not assign a blended position to each component. |
| WISE 0855 | Fast-moving brown dwarf | Use the NASA challenge-provided coordinates: RA `08h 55m 10.831s`, Dec `−07° 14′ 42.53″`; test coordinate parsing, actual archive coverage, timestamped frame sequence, and capability gates. |
| RW Aurigae | Variable-star target | Inspect actual observation times and photometric suitability; test variability only if compatible calibrated measurements and required uncertainty/quality data are available. |
| R Draconis | Variable-star target | Inspect actual observation times and photometric suitability; test variability only if compatible calibrated measurements and required uncertainty/quality data are available. |

For Barnard's Star, Epsilon Indi B, RW Aurigae, and R Draconis, do not hard-code coordinates from memory or infer them from a search label. Resolve each through an authoritative object resolver/catalogue, record the resolved identifier, coordinate source, reference epoch and any relevant proper-motion/component handling, then include that provenance in the target fixture. Epsilon Indi B requires component disambiguation before test expectations are set.

For every target, record the query parameters, current archive/release inventory, returned observation/product IDs, actual acquisition timestamps, compatible epoch count, channel/wavelength and detector-array availability, relevant FITS/WCS/units/flags/variance/PSF/contamination checks, cutout centering, and the resulting capability state. Execute only methods whose prerequisites pass.

A target test passes when HELIOS-X queries and reports the verified data and capability state correctly. It does not require multi-epoch coverage or a positive moving-object/variability detection. One valid observation may be displayed while multi-epoch analysis is reported as unavailable; a successful empty query must remain distinct from a provider failure; missing or inadequate metadata must block only the analyses that depend on it.

## 14. Methodology Governance

Scientific methods and decision policies must be versioned and reviewed when their implementation, assumptions, or applicable data products change.

For each operational method, maintain:

- Method identifier and version.

- Intended scientific purpose and supported data products.

- Required inputs and metadata.

- Algorithm and uncertainty method.

- Assumptions and limitations.

- Acceptance criteria and validation evidence.

- Known failure conditions.

- Output schema and status behavior.

Changes to methods, thresholds, or candidate-ranking policies must be documented. Where a change could affect scientific conclusions, previously generated results must remain identifiable, and reanalysis must create new result versions.

No method should be treated as validated merely because it executes without errors. Validation requires evidence that the implementation behaves appropriately for its intended scientific use.

## 15. Cross-Document Alignment and Method Ownership

This document defines scientific behavior and validation requirements. It does not replace the responsibilities of the other HELIOS-X specifications:

- `PRD.md` owns product scope, user journeys, P0/P1 boundaries, and product acceptance requirements.
- `SYSTEM_ARCHITECTURE.md` owns service boundaries, data flows, Explorer/Investigation separation, and the deterministic scientific-engine boundary.
- `DATA_SOURCE_SPEC.md` owns archive/source selection, release policy, retrieval interfaces, normalized product metadata, frame-manifest details, and provider-specific semantics.
- `INVESTIGATION_WORKFLOW.md` owns investigation execution order, task dependencies, orchestration outcomes, and evidence-gate flow.
- `AGENT_SPECIFICATIONS.md` owns the responsibilities and contracts of the eight logical agents; agents interpret and coordinate but do not invent or directly author scientific measurements.
- `API_SPECIFICATION.md` owns request/response contracts, and `DATABASE_SCHEMA.md` owns persistence representation and lineage.
- `TESTING_AND_ACCEPTANCE.md` owns the software-level test inventory; this document defines the scientific validity criteria those tests must enforce.

When a source product, release, or software implementation changes, update the relevant data-source and implementation contracts and revalidate every affected scientific method. Do not silently change scientific thresholds, candidate-ranking policies, coordinate conventions, or method assumptions in only one document. A method remains operationally unvalidated until its implementation has passed the documented tests on suitable controlled and/or authoritative data.

The integrated product boundary is mandatory: public image exploration is independently usable and does not implicitly launch an investigation; a scientific investigation begins only through an explicit request and must revalidate the selected evidence. The scientific engine must never upgrade visual appearance, an agent interpretation, or an unmatched catalogue result into a measured fact or confirmed discovery.
