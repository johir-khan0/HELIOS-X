# HELIOS-X — Data Source Specification

**Document:** `DATA_SOURCE_SPEC.md`  
**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Version:** 2.0 — Integrated SPHEREx Explorer + Agentic Investigation  
**Status:** Proposed implementation specification; archive access and scientific capabilities require product-level verification  
**Last reviewed:** 2026-11-01  
**Primary mission:** NASA SPHEREx  
**Primary archive:** NASA/IPAC Infrared Science Archive (IRSA)  
**Primary public use case:** Explore a user-selected sky region through actual SPHEREx images from distinct observation times  
**Integrated advanced use case:** Promote a selected region or source into a reproducible agentic scientific investigation

---

## Document Control and Authority

This specification defines how HELIOS-X discovers, retrieves, validates, displays, compares, and records SPHEREx observations and approved auxiliary astronomical data. It also defines the boundary between the public image-exploration experience and the deeper scientific investigation workflow.

This document is aligned with the combined product defined in `PRD.md` and the architecture in `SYSTEM_ARCHITECTURE.md`.

### Source-of-truth hierarchy

When requirements conflict, apply the following authority order:

1. **Official mission/archive product documentation** controls what a data product contains, its release status, file structure, calibration, units, flags, and supported access methods.
2. **`SCIENTIFIC_METHODOLOGY.md`** controls scientific assumptions, algorithms, registration acceptance, uncertainty, and interpretation of measurements.
3. **`PRD.md`** controls user-facing product requirements, user journeys, and MVP priorities.
4. **`SYSTEM_ARCHITECTURE.md`** controls component boundaries, data flow, and Explorer/Investigation separation.
5. **`API_SPECIFICATION.md`** controls exact public endpoint paths, payloads, status codes, and response contracts.
6. **`DATABASE_SCHEMA.md`** controls persistence entities, relationships, constraints, and migrations.
7. **`BACKEND.md`** controls backend implementation guidance within the contracts above.
8. **`FRONTEND.md` and `UI_UX_SPECIFICATION.md`** control presentation and interaction behavior, but cannot redefine scientific or archive semantics.
9. **`TESTING_AND_ACCEPTANCE.md`** defines the validation and release gates for supported capabilities.

This document must not invent archive endpoints, undocumented product fields, release identifiers, observation epochs, detector mappings, or scientific thresholds. If official documentation does not establish a fact, mark it `UNVERIFIED`, `UNKNOWN`, or `NOT_AVAILABLE` and record the verification task.

### Requirement language

- **MUST:** required for the NASA-facing core Explorer or for scientific/data integrity.
- **SHOULD:** strongly recommended; omission requires a documented reason.
- **MAY:** optional capability that can be added when the data, method, and implementation are validated.
- **MUST NOT:** prohibited behavior.

A documented feature is not an implemented feature. A retrieved file is not automatically a valid scientific observation, and a valid observation is not automatically sufficient for every analysis.

---

## 1. Product Context and Data Objectives

HELIOS-X has two integrated product capabilities:

1. **SPHEREx Sky Image Explorer — primary public-facing experience.** A visitor supplies Right Ascension (RA) and Declination (Dec), selects a supported field of view (FOV), discovers actual SPHEREx observations covering the requested region, and browses compatible images in acquisition-time order using individual frame navigation, a timeline, and video-like playback.
2. **Agentic Scientific Investigation Engine — advanced analysis experience.** A user explicitly promotes a selected region, observation set, or source into an investigation. The workflow validates the data, performs supported deterministic analyses, coordinates bounded AI agents, preserves evidence and provenance, evaluates alternatives, and prepares findings for human review.

The Explorer must work independently. A visitor must be able to retrieve and inspect available images without starting an agentic investigation, creating a candidate, or running a scientific analysis.

The Investigation Engine must be able to consume a controlled snapshot of selected Explorer inputs, but it must revalidate the referenced products and parameters before execution. Changing the currently selected frame in the browser must not silently change an already-created investigation.

### 1.1 NASA challenge requirements represented in this specification

The NASA Space Apps 2026 challenge statement, **“Planet X and SPHEREx,”** asks for a public-facing web tool that:

- accepts a sky position using RA and Dec;
- displays images of that sky patch from SPHEREx;
- makes images from different times easy to inspect as a short video-like sequence;
- helps users notice possible positional or brightness changes;
- is tested using Barnard's Star, Epsilon Indi B, WISE 0855, RW Aurigae, and R Draconis.

The challenge also identifies optional enhancements:

- make useful use of the 102 near-infrared spectral channels;
- consider combining data associated with each of the six detector arrays to improve signal-to-noise, when scientifically valid;
- generate spectra on demand;
- add or subtract images from different times;
- allow users to choose FOV;
- overlay information from Gaia or WISE;
- show metadata such as contamination flags;
- support coordinate parameters in the URL for embedding and sharing;
- evaluate astronomy tools such as Astropy and the Aladin sky atlas.

**Priority resolution:** the coordinate-driven, time-ordered SPHEREx image viewer is the core NASA deliverable. Spectral extraction, detector-array combination, image arithmetic, auxiliary catalogue overlays, and other advanced analysis are optional extensions. They must not delay a working core viewer or be presented as available until their input products and methods have been validated.

The challenge page includes a Resources area that may be updated. The team MUST revisit the official challenge page and its Resources tab during implementation and before final submission. A resource appearing on that page is a pointer for evaluation, not proof that it is a mission data API or an endorsed HELIOS-X integration.

### 1.2 Main data objectives

- Retrieve actual SPHEREx spectral-image products from IRSA through documented access routes.
- Discover which products and observation times genuinely cover a requested sky region.
- Produce small, WCS-aware cutouts or display derivatives where the source product and archive tools permit.
- Create a versioned frame manifest with stable observation/product references, actual acquisition timestamps, and controlled image-artifact references.
- Preserve spectral-channel, detector-array, release, calibration, quality, variance, PSF, contamination, and coordinate metadata when supplied.
- Support an image timeline without confusing observation time with wavelength, spectral channel, retrieval time, processing time, or release date.
- Keep original scientific products separate from browser previews and derived comparison images.
- Provide traceable, replaceable adapters for auxiliary catalogues and mission tools such as Gaia and WISE.
- Feed selected Explorer data into the Agentic Investigation Engine without allowing agents to fabricate or reinterpret source records.

### 1.3 Non-goals of this document

This document does not:

- claim that every requested region has multiple usable SPHEREx epochs;
- promise that each named NASA test target is covered by multiple compatible public images;
- define exact HTTP endpoint names owned by `API_SPECIFICATION.md`;
- define database columns owned by `DATABASE_SCHEMA.md`;
- define scientific thresholds or equations owned by `SCIENTIFIC_METHODOLOGY.md`;
- establish that HELIOS-X has discovered Planet X, another planet, or any new object;
- require the entire SPHEREx survey to be downloaded;
- require every optional NASA enhancement in the first release;
- treat Gaia, WISE, SIMBAD, VizieR, JPL Horizons, or Aladin as SPHEREx data feeds.

---

## 2. Data and Instrument Concepts That Must Remain Separate

SPHEREx combines spectral and temporal survey information. Correct handling requires separate data dimensions rather than treating all image-like records as interchangeable.

### 2.1 Core dimensions

| Dimension | Meaning | Correct use in HELIOS-X |
|---|---|---|
| Sky position | Celestial coordinate and selected region | Query center, cutout center, overlay alignment |
| Observation epoch | Actual time at which a product's observation was acquired | Timeline ordering, time-series comparison, motion/variability analysis |
| Spectral channel | A wavelength sample or channel defined by product metadata | Channel selector, spectrum construction, compatible wavelength comparison |
| Broad wavelength band | Instrument/product wavelength interval or detector spectral coverage | High-level band selection and data grouping |
| Detector array | Physical detector identity associated with a measurement/product | Array-specific validation and optional array-specific coaddition |
| Product release | Processing/data release such as QR2 or QR3 | Reproducibility, calibration compatibility, release-aware query |
| Processing version | Pipeline/software version used to create the product | Calibration compatibility and provenance |
| Retrieval time | Time HELIOS-X queried or downloaded the product | Operational audit, cache policy; never a substitute for observation time |

### 2.2 102 spectral channels are not 102 observation epochs

The 102 SPHEREx channels represent spectral sampling across wavelength. They are not 102 independent dates or repeated observations of the same source.

A time-sequence viewer MUST order frames by validated acquisition time. A channel selector MUST remain a separate control or dimension. A user changing the selected channel must not cause the timeline to imply that time has advanced.

If a source has one observation containing multiple spectral channels but no second acquisition epoch, the Explorer may display its spectral information where supported, but it MUST NOT fabricate a multi-epoch playback sequence.

### 2.3 Six detector arrays are not interchangeable with channels or epochs

The NASA challenge mentions six detector arrays as a possible signal-to-noise enhancement. The instrument contains six physical detector arrays; the 102-channel spectral dimension and actual observation-time dimension remain separate.

The adapter MUST preserve the detector/array identity when it is available in the product metadata. HELIOS-X MUST NOT assume that:

- six detector arrays mean six observation epochs;
- six detector arrays mean 102 channels;
- any six array products can be directly pixel-added;
- array identifiers can be inferred from filenames without product documentation;
- arrays have identical PSF, calibration, wavelength response, units, or WCS.

The mapping between hardware arrays, wavelength ranges, spectral channels, and file extensions must be taken from the relevant instrument/product documentation for the selected release.

### 2.4 Six-month sky-survey cadence is not a per-target coverage guarantee

The challenge describes SPHEREx as mapping the whole sky approximately every six months. This is useful mission context, but it does not establish that every requested coordinate has two or more directly comparable images in the currently released archive subset.

For each query, HELIOS-X MUST determine actual returned observations, actual acquisition timestamps, coverage, product release, selected wavelength/channel, calibration compatibility, and quality. It MUST NOT fabricate missing epochs, interpolate a missing image as though it were observed, or assume uniform cadence.

### 2.5 Six broad wavelength bands

IRSA documentation describes six broad SPHEREx wavelength intervals, approximately:

| Band | Approximate wavelength interval | Approximate resolving power |
|---|---:|---:|
| 1 | 0.75–1.09 µm | R ≈ 39 |
| 2 | 1.10–1.62 µm | R ≈ 41 |
| 3 | 1.63–2.41 µm | R ≈ 41 |
| 4 | 2.42–3.82 µm | R ≈ 35 |
| 5 | 3.83–4.41 µm | R ≈ 112 |
| 6 | 4.42–5.00 µm | R ≈ 128 |

These are approximate reference ranges, not substitutes for product metadata. The effective wavelength and bandwidth can vary with detector position because SPHEREx uses a linear variable filter. The spectral WCS, channel maps, wavelength lookup tables, calibration products, and release-specific documentation are authoritative for a particular pixel/product.

Do not assign one fixed wavelength to an entire image unless the applicable product definition explicitly supports that representation.

---

## 3. Source Registry and External Resources

The source registry distinguishes operational data sources from mission background, scientific methods, user-interface libraries, catalogues, and challenge references. Their roles must not be conflated.

### 3.1 Official NASA challenge and competition resources

**NASA Space Apps Challenge 2026 — Planet X and SPHEREx**  
URL: <https://www.spaceappschallenge.org/2026/challenges/planet-x-and-spherex/>  
Role: Authority for the challenge's public-facing image-viewer objective, coordinate input, time-based image display, five required test targets, optional enhancements, and submission context. Review the challenge statement and the Resources tab directly; record the access date and any newly added materials. This page is not an astronomical data API.

**NASA Space Apps Challenge Project Submission Guide**  
URL: <https://www.spaceappschallenge.org/resources/project-submission-guide/>  
Role: Competition submission expectations and presentation guidance. It does not define SPHEREx file formats, calibration, or archive query semantics.

**NASA Space Apps Challenge 2026 General Page**  
URL: <https://www.spaceappschallenge.org/2026/>  
Role: Competition-wide context, dates, rules, and logistics. It is not an operational data source.

**Challenge Resources-tab policy**

- Revisit the challenge's Resources tab during each major integration milestone and before submission.
- Add newly supplied material to this registry only after checking its owner, scope, version/date, and relevance.
- Classify each resource as mission/product documentation, software/tooling, catalogue/data service, scientific literature, or competition guidance.
- Do not automatically promote a resource to an authoritative source merely because it is linked by the challenge page.
- Record resource title, exact URL, access date, and the implementation decision it informs.

### 3.2 Primary operational data source — SPHEREx at IRSA

**IRSA SPHEREx Mission and Archive Landing Page**  
URL: <https://irsa.ipac.caltech.edu/Missions/spherex.html>  
Role: Primary entry point for the SPHEREx archive, current products, science tools, release documentation, and programmatic interfaces. Follow current release-specific documentation for implementation details.

**SPHEREx Quick Release Overview**  
URL: <https://irsa.ipac.caltech.edu/data/SPHEREx/docs/overview_qr.html>  
Role: Quick Release product status, access options, data citation, and acknowledgement. Cross-check its current contents against the newer archive documentation because overview pages may not all update simultaneously.

**SPHEREx Archive User Guide**  
URL: <https://caltech-ipac.github.io/spherex-archive-documentation/>  
Role: Current guide for available products, archive discovery, downloading, tool usage, and how product documentation evolves.

**SPHEREx Data Products — IRSA Archive Documentation**  
URL: <https://caltech-ipac.github.io/spherex-archive-documentation/spherex-data-products/>  
Role: Release-specific product inventory, processing history, and available/expected higher-level products. Use this page together with the live IRSA archive to determine what is actually accessible.

**SPHEREx Explanatory Supplement and Product Documentation**  
URL: <https://irsa.ipac.caltech.edu/data/SPHEREx/docs/>  
Role: Authoritative product descriptions, FITS structures, extensions, calibration, quality flags, PSF, wavelength metadata, and release-specific limitations.

**SPHEREx Data Explorer**  
URL: <https://irsa.ipac.caltech.edu/applications/spherex/>  
Role: Interactive manual discovery and visual inspection of archive products. It is useful for validating queries and comparing application behavior. Production code should use a documented programmatic interface rather than automating the interactive page.

**IRSA Program Interfaces**  
URL: <https://irsa.ipac.caltech.edu/voapi.html>  
Role: Entry point for programmatic archive access, including Virtual Observatory services such as SIA and TAP where supported.

**IRSA SIA Interface**  
URL: <https://irsa.ipac.caltech.edu/ibe/sia.html>  
Role: Programmatic discovery of image products and their metadata. Validate the current table, collection, parameter, and response definitions before implementation; do not hard-code identifiers copied from an older release example.

**IRSA Astroquery Documentation**  
URL: <https://astroquery.readthedocs.io/en/latest/ipac/irsa/irsa.html>  
Role: Python access to IRSA services, including SIA query workflows where supported. Pin compatible dependency versions and use current SPHEREx examples.

**PyVO Documentation**  
URL: <https://pyvo.readthedocs.io/>  
Role: Python tools for Virtual Observatory protocols such as SIA/TAP. Use only when the current IRSA service contract supports the needed query.

**IRSA Cloud Data Access**  
URL: <https://irsa.ipac.caltech.edu/cloud_access/>  
Role: Cloud-based access to SPHEREx products. Current IRSA documentation lists the public SPHEREx bucket as `nasa-irsa-spherex` in `us-east-1`; verify current bucket/prefix, release, access policy, and transfer conditions before use. Cloud access is an alternative for appropriate bulk/cloud workflows, not a reason to download the full survey from an interactive browser request.

**AWS Open Data Registry — SPHEREx Quick Release**  
URL: <https://registry.opendata.aws/spherex-qr/>  
Role: Public cloud dataset registry and access information. IRSA documentation remains authoritative for mission-specific product semantics.

### 3.3 SPHEREx tools and notebooks

**SPHEREx Cutout Tool**  
URL: <https://irsa.ipac.caltech.edu/data/SPHEREx/docs/cutout_tool.html>  
Role: Extract spatial subsets from large multi-extension FITS images while preserving relevant structures and metadata. Cutouts are preferred for bounded-region development when they retain all fields needed by the selected analysis.

**SPHEREx Spectrophotometry Tool**  
URL: <https://irsa.ipac.caltech.edu/onlinehelp/spherex/spherex/sp.html>  
Role: Extract spectra using forced photometry at supplied source positions. It is not a blind source-identification tool. Treat its outputs as a distinct scientific product with its own inputs, assumptions, quality information, and validation.

**SPHEREx Mosaic Tool / Custom Mosaic documentation**  
Start from: <https://irsa.ipac.caltech.edu/Missions/spherex.html>  
Role: Create custom multiwavelength mosaics or selected-band image products where supported. Confirm current interface, product semantics, and output metadata before integration.

**SPHEREx Source Discovery Tool tutorial**  
URL: <https://caltech-ipac.github.io/irsa-tutorials/spherex-source-discovery-tool-demo/>  
Role: Reference workflow for identifying/characterizing significant sources in a user-defined region without positional priors. The existence of this tool does not by itself prove that the same algorithm, service interface, or thresholds are available to HELIOS-X. Review its method, inputs/outputs, access route, and scientific suitability before calling it from the application.

**SPHEREx Moving Object Search Tool (MOST)**  
URL: <https://irsa.ipac.caltech.edu/applications/MOST/>  
Instructions: <https://irsa.ipac.caltech.edu/applications/MOST/instructions.html>  
Role: Search supported IRSA image archives at predicted locations of a known Solar System object using its ephemeris/orbital elements. It can support known-object coverage checks or precovery-style searches. It is not a blind detector for an unknown Planet X, and an image covering a predicted location is not, by itself, a confirmed detection.

**SPHEREx Introduction Notebook**  
URL: <https://caltech-ipac.github.io/irsa-tutorials/spherex-intro/>  
Role: Practical reference for inspecting SPHEREx spectral images, archive metadata, and products.

**SPHEREx Cutout Notebook**  
Accessible from the current SPHEREx archive documentation and cutout page.  
Role: Reference for requesting and validating cutouts programmatically.

**SPHEREx PSF-in-Cutouts Notebook**  
Accessible from the current SPHEREx archive documentation.  
Role: Reference for preserving/interpreting PSF information in derived cutouts.

### 3.4 Mission and instrument context

**NASA SPHEREx Mission**  
URL: <https://science.nasa.gov/mission/spherex/>  
Role: Mission purpose, survey context, launch/background information, and official public updates. It is not the programmatic image-download interface.

**NASA — How SPHEREx Will Share Its All-Sky Map / Universe Map**  
URL: <https://science.nasa.gov/open-science/spherex-universe-map/>  
Role: Public data-release and open-science context. It explains the public archive and the importance of releasing data and processing information.

**NASA SPHEREx by the Numbers**  
URL: <https://science.nasa.gov/blogs/spherex/2025/03/11/nasas-spherex-by-the-numbers/>  
Role: Mission overview and survey-cadence context. It is not a substitute for actual archive acquisition timestamps.

**JPL SPHEREx Mission**  
URL: <https://www.jpl.nasa.gov/missions/spherex/>  
Role: Mission and science context, including planned repeated all-sky survey behavior.

**SPHEREx Instrument Description — Caltech**  
URL: <https://spherex.caltech.edu/page/instrument>  
Role: Instrument design, field of view, detector arrays, wavelength coverage, and instrument-level context. Product-level release documentation still controls individual file interpretation.

**SPHEREx Data Products — Caltech**  
URL: <https://spherex.caltech.edu/page/data-products>  
Role: Product roadmap and expected data-delivery schedule. Roadmaps are plans, not proof of current release availability.

**SPHEREx Science Products — Caltech**  
URL: <https://spherex.caltech.edu/page/science-products>  
Role: Higher-level product roadmap, including future catalogues. Check current IRSA availability before using any planned product.

### 3.5 Astronomy software and visualization tools referenced by the challenge

**Astropy — Main Documentation**  
URL: <https://docs.astropy.org/en/stable/>  
Role: Core Python astronomy utilities. Relevant modules include:

- `astropy.coordinates` for sky coordinate parsing, transformation, and separation;
- `astropy.time` for time formats, scales, and conversions;
- `astropy.units` for unit-safe operations;
- `astropy.io.fits` for FITS inspection and reading;
- `astropy.wcs` for world-coordinate-system transformations.

Use Astropy for well-defined scientific/data operations; do not assume that installing Astropy alone validates an image-registration or motion algorithm.

**Astropy Coordinates**  
URL: <https://docs.astropy.org/en/stable/coordinates/>  
Role: RA/Dec parsing, coordinate frames, epoch-aware transformations where supported, and coordinate calculations.

**Astropy Time**  
URL: <https://docs.astropy.org/en/stable/time/>  
Role: Timestamp parsing, time scales, formats, and conversions. The original time scale and source field must still be preserved.

**Astropy FITS**  
URL: <https://docs.astropy.org/en/stable/io/fits/>  
Role: FITS reading and metadata inspection.

**Astropy WCS**  
URL: <https://docs.astropy.org/en/stable/wcs/>  
Role: Pixel-to-sky transformations and WCS metadata handling.

**Aladin Lite Documentation — CDS Strasbourg**  
URL: <https://aladin.cds.unistra.fr/AladinLite/doc/>  
API documentation: <https://cds-astro.github.io/aladin-lite/>  
Role: Embeddable browser-based sky atlas for sky navigation, contextual survey visualization, and catalogue/footprint overlays. Aladin Lite can visualize supported survey layers and overlay tabular/footprint data. It may be evaluated as a component of the sky-map experience.

**Aladin integration boundary:** Aladin Lite is a sky-atlas/overlay tool, not a substitute for HELIOS-X's SPHEREx-specific epoch manifest, frame-by-frame time navigation, or scientifically validated image subtraction. The Explorer may use Aladin for navigation/context and a dedicated image-frame viewer for retrieved SPHEREx cutouts. Respect the Aladin attribution/logo/link terms and pin a tested release.

**Astroquery**  
URL: <https://astroquery.readthedocs.io/>  
Role: Python interfaces to astronomical archives and catalogues. Each service adapter must be documented and tested independently.

**Terminology note:** the challenge wording supplied for this project says “Astrophy tools.” This specification interprets that reference as **Astropy**, the established Python astronomy library documented above. Do not add or depend on a separate package called “Astrophy” unless the official challenge Resources tab identifies a distinct tool by that exact name.

### 3.6 Auxiliary data sources suggested by NASA — Gaia and WISE

These are auxiliary context/cross-match sources. They do not replace SPHEREx and must not be treated as SPHEREx observations.

**ESA Gaia Archive**  
URL: <https://gea.esac.esa.int/archive/>  
Role: Gaia astrometry, parallax, proper motion, photometry, radial velocities, and other catalogue information where available. Gaia is an ESA mission/data archive, not a NASA archive. Record the exact data release/product, query, source ID, astrometric reference epoch, units, and uncertainty fields. Gaia DR3 is a known public release; any newer release or focused data product must be verified as public and stable before use. Do not assume a planned/in-preparation release is production-ready.

**Gaia DR3 Documentation**  
URL: <https://gea.esac.esa.int/archive/documentation/GDR3/>  
Role: Data model, release-specific fields, known issues, cross-match documentation, and data access details.

**IRSA WISE and NEOWISE Mission Page**  
URL: <https://irsa.ipac.caltech.edu/Missions/wise.html>  
Role: Authoritative IRSA entry point for WISE/NEOWISE images, catalogues, programmatic interfaces, coadds, and time-series tools.

**IRSA WISE Image Service**  
URL: <https://irsa.ipac.caltech.edu/onlinehelp/wise/wise/overview.html>  
Role: Access WISE/NEOWISE image products and release documentation. Select the relevant release based on the scientific use and actual observation time.

**IRSA WISE Catalog Search**  
Start from: <https://irsa.ipac.caltech.edu/Missions/wise.html>  
Role: Search tabular WISE/NEOWISE catalogue products.

**WISE/NEOWISE Time Series Tool**  
Start from: <https://irsa.ipac.caltech.edu/Missions/wise.html>  
Role: Explore WISE/NEOWISE time-domain photometry and associated images. This is an auxiliary time series with its own epochs and calibration; it must not be merged into a SPHEREx timeline as if it were the same instrument/product.

**WISE/NEOWISE Coadder**  
Start from: <https://irsa.ipac.caltech.edu/Missions/wise.html>  
Role: Create WISE/NEOWISE mosaics from supported mission phases. Preserve mission phase, observation windows, band, calibration, and coadd provenance.

**WISE/NEOWISE release context:** IRSA lists the WISE cryogenic, 3-band, post-cryogenic, and NEOWISE Reactivation data sets; its mission page lists the NEOWISE Reactivation mission through 2024-08-01 and the final 2024 data release. Do not claim that NEOWISE is still collecting new observations. Use the archive's actual release inventory and observation timestamps.

**Overlay boundary:** Gaia and WISE overlays are contextual layers. They must be rendered separately from SPHEREx pixel values, with their own source, release, coordinate epoch, units, and provenance. Cross-matching for a scientific conclusion remains subject to the methodology and evidence gates.

### 3.7 Object-name resolution and Solar System ephemerides

**SIMBAD — CDS Strasbourg**  
URL: <https://simbad.cds.unistra.fr/simbad/>  
Role: Optional object-name resolution and astronomical identifier cross-reference. It is not an image archive and is not proof of object classification. Store the resolver, identifier, returned coordinates, coordinate frame/epoch if supplied, and resolution timestamp.

**VizieR — CDS Strasbourg**  
URL: <https://vizier.cds.unistra.fr/>  
Role: Catalogue discovery and access. A catalogue must be selected by a specific scientific purpose, release, data model, and uncertainty policy before integration.

**JPL Horizons**  
URL: <https://ssd.jpl.nasa.gov/horizons/>  
Role: Ephemeris generation for supported known Solar System objects. Used in conjunction with tools such as IRSA MOST when the object has an accepted name/designation or orbital solution. It cannot produce a verified ephemeris for a hypothetical object with no defined orbit.

**MOST/Horizons limitation:** a predicted known-object position can guide archive searches; it does not prove the object is present in the returned image. It must not be used as a blind Planet X discovery mechanism.

### 3.8 Planet X scientific context

**NASA — Hypothetical Planet X / Planet Nine**  
URL: <https://science.nasa.gov/solar-system/planet-x/>  
Role: Responsible scientific/public communication. Planet X/Planet Nine remains hypothetical and has not been confirmed as discovered. Use this reference to avoid overstating candidate results.

**Scientific literature — SPHEREx image/spectrophotometry pipeline**  
URL: <https://arxiv.org/abs/2511.15823>  
Role: Technical context for the image and spectrophotometry pipeline. Review the current paper version and use official release documentation for operational file semantics.

**Scientific literature — Planet Nine search context**  
DOI: <https://doi.org/10.3847/1538-3881/ad24e9>  
Role: Context for previous Planet Nine search methods and survey limitations. It is not an operational SPHEREx interface.

**Scientific literature — SPHEREx spectra for nearby brown dwarfs**  
URL: <https://arxiv.org/abs/2607.00543>  
Role: Example of SPHEREx spectral applications to brown dwarfs. Do not assume that its model or analysis method directly applies to a Planet X candidate.

### 3.9 Sources that must not be registered as SPHEREx feeds

NASA-hosted content is not automatically relevant to astronomical observations. The following types of resources are not SPHEREx image/catalogue data sources:

- NASA Physical Sciences Informatics records for unrelated experiments;
- combustion-science, SOFIE, SAFFIRE, or microgravity combustion datasets;
- unrelated NASA Technical Reports Server items;
- papers on unrelated imaging domains.

They may be useful for unrelated research, but MUST NOT be registered as SPHEREx observation adapters or ingested as astronomical evidence for this product.

---

## 4. SPHEREx Product Availability and Release Policy

### 4.1 Current release policy

As reviewed on 2026-11-01, IRSA's archive documentation reports that QR3 spectral-image distribution began in September 2026. QR3 includes improved PSF and other calibration information intended to improve photometric reliability. QR2 remains relevant for reproducibility and cases where requested data are not available in QR3. QR1 was formally retired in February 2026 and must not be selected as a current default.

**Default policy:**

1. Prefer QR3 when the requested product is available and scientifically appropriate.
2. Use QR2 only when the investigation explicitly requires it, a required product is not available in QR3, or a documented compatibility reason applies.
3. Do not silently combine QR2 and QR3 measurements in one analysis.
4. If a cross-release comparison is scientifically necessary, use a documented and validated calibration/compatibility method, retain both release identities, and disclose the limitation.
5. Record the exact release and processing version in every retrieval, observation, artifact, measurement, and report lineage where applicable.
6. Reprocessing or a release change creates a new identifiable run/result version; it does not silently overwrite previous evidence.
7. Re-check live archive availability at runtime. A release page or roadmap does not prove that a specific coordinate, week, channel, product, or file is available.

### 4.2 Product families

| Product family | Role | Availability policy |
|---|---|---|
| Calibrated spectral images | Primary source for Explorer frames and many initial analyses | Use current public IRSA releases; validate each product |
| Spatial cutouts | Bounded subregion of a parent spectral image | Use official cutout service or a validated local process; preserve parent link |
| Multiwavelength mosaics | Custom image assembled from selected bands/channels | Use only when current tool/product semantics fit the task |
| Spectrophotometry outputs | Spectra derived through an extraction/forced-photometry workflow | Treat as a derived scientific product with its own method and provenance |
| All-sky spectral cubes | Higher-level data products containing spectral channels | Availability is release-dependent; check current archive inventory |
| High-reliability source catalogues | Higher-level source catalogue products | Do not assume a roadmap date equals actual public availability |
| Focused Solar System object catalogues | Higher-level catalogue for Solar System science | Use only after current public release and field semantics are verified |

### 4.3 Higher-level product roadmap

Caltech/SPHEREx roadmap pages describe future data products including all-sky cubes, high-reliability source catalogues, and focused Solar System products. Roadmap dates can change and are not guarantees. As of this specification's review date, the archive and current product listings—not an earlier schedule—determine actual availability.

Before a feature depends on a higher-level product, the implementation owner MUST:

- verify that the product is publicly listed by IRSA;
- retrieve the current documentation and data model;
- verify its release/version, fields, units, calibration, and access method;
- retrieve a small representative product;
- test the feature's scientific assumptions;
- update this document and the relevant API/database/testing specifications.

### 4.4 Release compatibility

Two products are not compatible merely because they have the same filename pattern or cover the same coordinate. Compatibility checks may need to consider:

- release and processing version;
- instrument/detector identity;
- observation time and scan/pointing context;
- spectral channel, wavelength, bandwidth, and response;
- calibration state and units;
- WCS, orientation, projection, pixel scale, and dimensions;
- PSF and background behavior;
- masks, pixel flags, variance, and contamination information;
- product-specific changes documented between releases.

Compatibility is operation-specific. Two frames may be suitable for visual browsing but not for subtraction, coaddition, photometry, or motion estimation.

---

## 5. Primary Archive Access Strategy

### 5.1 Replaceable adapter

Implement a dedicated `SpherexIRSAAdapter` behind a typed interface in the backend data-source module, aligned with the canonical location `backend/src/helios_x/data_sources/` defined by `SYSTEM_ARCHITECTURE.md`. Archive-specific URLs, query formats, collection names, response parsing, and retry logic must remain inside the adapter.

The root-level `agents/` package may request approved retrieval operations through typed backend tools, but agents must not duplicate the IRSA adapter, call undocumented archive routes directly, or own product persistence. The root-level `database/` directory is not a second runtime data store; runtime sessions, repositories, and migrations remain in the backend as specified by the system architecture.

The adapter should expose logical operations such as:

- `discover_observations(query)`;
- `retrieve_product(product_reference)`;
- `request_cutout(product_reference, region, options)`;
- `validate_product(product_reference_or_artifact)`;
- `retrieve_product_metadata(product_reference)`;
- `get_supported_capabilities()`.

These are conceptual method names. They do not prescribe public API endpoint paths. Exact public routes belong to `API_SPECIFICATION.md`.

### 5.2 Choose one verified primary programmatic path

For the first live integration:

1. Prefer a documented IRSA programmatic interface capable of returning the required spectral-image records and metadata.
2. Evaluate SIA or a documented Astroquery/PyVO workflow against the current release and required query filters.
3. Choose one primary route for the first implementation and document why it was selected.
4. Preserve the ability to add another route behind the adapter if the archive changes or a workload needs it.
5. Do not code against guessed collection names or stale sample request parameters.
6. Use the SPHEREx Data Explorer for manual verification when useful, but do not automate its browser interface as the production data API.
7. Use cloud/direct archive access only after confirming current path conventions, access policy, and the workload's suitability.

IRSA SIA query results may expose sky position, product/observation identifiers, time bounds, wavelength bounds, resolution, access URLs, and estimated file size. These response fields and names are examples, not a frozen HELIOS-X contract. Parse the live response schema and record which filters were actually applied.

### 5.3 Search inputs

A sky-region query may use the following inputs when supported by the selected IRSA interface:

- RA and Dec;
- coordinate frame and reference epoch where required;
- FOV/region geometry (for example, circular radius or rectangular width/height);
- observation time range;
- product family and release;
- spectral channel, band, or wavelength selection;
- instrument/detector/array selection where exposed;
- result/frame limit and data-size budget;
- requested output type (metadata only, source product, cutout, or display preview).

The adapter must distinguish a filter applied by the archive from a filter applied locally after retrieval. It MUST NOT claim the archive applied a filter if the selected interface does not support it.

### 5.4 Metadata-first retrieval

For interactive Explorer requests:

1. validate the query;
2. run a bounded metadata search;
3. return observation availability and coverage information;
4. retrieve only the selected products or cutouts required by the viewer;
5. generate display derivatives only when needed;
6. retain a link to the original product and the exact query.

Do not download full products for every result when a validated cutout is sufficient. Do not download the entire survey for a normal coordinate query.

### 5.5 Cutouts and download size

SPHEREx spectral-image files may be large multi-extension FITS products. Use IRSA's cutout service where appropriate and supported. A cutout must:

- preserve the parent product reference and release;
- retain or link all extensions needed for the intended operation;
- preserve or correctly update WCS;
- document whether it is archive-generated or generated locally;
- record center coordinate, region size, pixel scale, output dimensions, and processing version;
- retain quality/flag, variance, PSF, and spectral metadata or explicitly state which are not included;
- be validated after retrieval.

A cutout that drops a necessary uncertainty or quality extension is not automatically suitable for scientific analysis even if the image plane displays correctly.

### 5.6 Rate limits, retries, and bulk jobs

- Do not invent an archive quota or rate limit.
- Check current IRSA terms and service documentation.
- Apply timeouts and bounded exponential backoff only for transient failures.
- Respect `Retry-After` when supplied.
- Do not retry permanent validation, unsupported-operation, or authorization failures without changed conditions.
- Use documented cloud or bulk-transfer routes for large jobs rather than unbounded downloads from interactive endpoints.
- Bound query area, frame count, channel count, concurrent requests, and total transfer size using configuration derived from observed service capacity and deployment requirements.

---

## 6. Coordinate, Target, and Field-of-View Handling

### 6.1 RA and Dec input

The Public Explorer MUST allow users to specify RA and Dec. Input formats may include:

- decimal degrees;
- sexagesimal RA (hours, minutes, seconds);
- sexagesimal Dec (degrees, arcminutes, arcseconds).

The frontend should show the expected format and units. The backend must parse and validate the selected format explicitly. Never infer whether a numeric RA is degrees or hours from the value alone.

### 6.2 Canonical coordinate representation

The backend should normalize valid coordinates into a documented canonical internal representation, typically decimal degrees, while retaining:

- original user input;
- parsed numeric coordinates;
- coordinate frame;
- equinox/reference epoch where relevant;
- source/resolver if coordinates were obtained from a name;
- transformation history if coordinates were transformed.

Use Astropy or another validated coordinate implementation for parsing and transformations. Coordinate transformations must not silently discard epoch/frame information.

### 6.3 Validation

- RA must be finite and normalized to the supported range, with wraparound behavior tested.
- Dec must be finite and within the valid celestial latitude range.
- Coordinate units and representation must be explicit.
- Invalid, ambiguous, or unsupported coordinate frames must return a useful validation error.
- Query geometry must handle RA wraparound, high declinations, and polar edge cases through tested spherical-coordinate behavior.
- Preserve the original user input for reproducibility.

### 6.4 Target-name resolution

The Explorer may offer known-object lookup as a convenience. If implemented:

1. send the name to a documented resolver/catalogue;
2. handle aliases and ambiguous matches explicitly;
3. show the proposed resolved object and coordinates to the user;
4. store resolver name/version, identifier, coordinates, frame/epoch, uncertainty/source reference, and resolution timestamp;
5. require user confirmation when multiple objects or uncertain identifications exist.

A name resolver's result is a coordinate cross-reference, not a scientific proof of object identity in a particular SPHEREx image.

### 6.5 Field of view

The NASA challenge explicitly asks that users be able to choose FOV. HELIOS-X MUST provide a user-facing FOV control within server-defined supported bounds.

The contract should distinguish:

- center coordinate;
- region geometry (circle, rectangle, or supported polygon);
- width/height or radius;
- angular units;
- requested output pixel dimensions or pixel scale, where appropriate.

The backend must enforce configured area and data-size limits. The UI must display when a requested FOV is capped, unsupported, or unavailable. Do not imply that all FOVs have the same resolution, sensitivity, or coverage.

### 6.6 URL-addressable queries

To support embedding and sharing, the Public Explorer SHOULD encode the minimal reproducible viewer query in URL parameters, such as RA, Dec, FOV, selected time range, and selected channel/band. The exact parameter names and canonical URL shape belong in `API_SPECIFICATION.md` and `FRONTEND.md`.

A shareable URL must not expose private investigation IDs, access tokens, local file paths, or administrator-only data. On page load, validate query parameters exactly as if they had been entered in the UI. Invalid parameters must fail safely.

---

## 7. Observation Discovery, Frame Manifest, and Time-Based Viewer Data

### 7.1 Observation versus frame

An **observation** is an archive/product record with acquisition metadata. A **viewer frame** is a displayable image/cutout representation derived from a specific product/observation and a selected spectral/channel configuration.

One source product may expose multiple spectral channels or planes. These must not automatically become separate time frames. A frame record must identify the actual acquisition time separately from its spectral/channel identity.

### 7.2 Frame-set discovery

For a user query, the adapter/service must determine:

- all returned products that actually overlap the requested region;
- their real observation identifiers and acquisition times;
- selected product release and processing version;
- which wavelengths/channels are represented;
- which detector-array information is supplied;
- whether WCS and cutout geometry are valid;
- whether multiple observations are distinct in acquisition time;
- whether frames are compatible for the requested display or comparison mode;
- which quality, contamination, mask, variance, calibration, and PSF details are available.

### 7.3 Frame ordering

- Sort the timeline by actual acquisition time from validated metadata.
- Do not sort by filename, retrieval order, release date, or channel number.
- Preserve the original timestamp representation and time scale.
- If only date-level precision is available, represent the precision honestly.
- If times are missing or ambiguous, do not invent a precise order; mark ordering uncertainty or exclude the frame from a time-dependent operation.
- Show coverage gaps where no frames exist. Do not create interpolated images.

### 7.4 Versioned frame manifest

The backend should provide a versioned frame manifest for the selected query. At minimum, it should support these logical fields where available:

| Field | Meaning |
|---|---|
| `frame_id` | Stable HELIOS-X frame reference |
| `observation_id` | Archive observation identifier |
| `product_id` | Archive product identifier |
| `source_archive` | Data archive/service name |
| `mission` / `instrument` | Mission/instrument identity |
| `product_family` | Spectral image, cutout, cube, or other documented family |
| `release` | QR2, QR3, or actual applicable release |
| `processing_version` | Product pipeline version |
| `observation_time` | Actual acquisition timestamp |
| `time_scale` / `time_format` | Time scale and representation when known |
| `retrieved_at` | HELIOS-X retrieval timestamp |
| `coordinate_center` | Validated region center |
| `coordinate_frame` / `reference_epoch` | Coordinate interpretation |
| `fov` | Actual cutout/region geometry |
| `channel_id` / `wavelength` | Spectral dimension; not a time index |
| `detector_array_id` | Detector identity where supplied |
| `image_dimensions` | Frame pixel dimensions |
| `pixel_scale` | Pixel scale where supplied/derived validly |
| `unit` | Pixel/intensity unit |
| `wcs_reference` | WCS metadata or controlled reference |
| `quality_flags_reference` | Flag/quality-plane reference where available |
| `contamination_metadata` | Contamination flags/indicators where supplied |
| `variance_reference` | Variance data reference where available |
| `psf_reference` | PSF metadata/reference where relevant |
| `original_product_reference` | Immutable source product link/reference |
| `display_artifact_reference` | Controlled browser-display derivative |
| `checksum` | File/artifact checksum if computed |
| `retrieval_status` | Explicit retrieval outcome |
| `warnings` | Known limitations or incomplete metadata |

The full schema is owned by `API_SPECIFICATION.md` and persistence representation by `DATABASE_SCHEMA.md`. Missing metadata must remain missing/unknown; do not invent defaults.

### 7.5 Viewer playback support

The data layer must provide valid, controlled image artifacts and a manifest. The frontend owns:

- play/pause;
- previous/next frame;
- timeline scrubbing;
- playback interval/speed controls;
- frame counter;
- active observation timestamp;
- coverage-gap presentation;
- optional blink mode between selected epochs;
- optional side-by-side comparison.

A server-generated video file is not required for the core NASA viewer. Individual frames must remain accessible so users can inspect exact timestamps and metadata.

The viewer must not silently skip invalid frames, treat an error screen as a real frame, or fill gaps with synthetic imagery.

---

## 8. FITS, Image Planes, Quality Metadata, and Artifact Handling

### 8.1 Original scientific products

The original SPHEREx multi-extension FITS product is the scientific source of truth. Keep it immutable in controlled storage or retain a durable archive reference with checksum/identity when the product is not locally mirrored.

A browser preview is a derivative. It must never replace the source FITS product in scientific processing or provenance.

### 8.2 Common extensions and semantics

Current SPHEREx tool documentation describes image products with extension roles such as:

- `IMAGE`: pixel flux/intensity values; a documented example unit is MJy/sr;
- `FLAGS`: pixel-level quality flags, including problematic pixels and source/background masking information;
- `VARIANCE`: per-pixel variance information;
- `ZODI`: modelled zodiacal-light background; the documented image product may not subtract this model from the `IMAGE` plane;
- `PSF`: position-dependent point-spread-function information;
- `WCS-WAVE`: spectral wavelength/bandwidth lookup information used by the spectral WCS in documented multi-extension products.

This is a guide, not a fixed schema for all releases. The selected release's FITS supplement is authoritative. Validate actual extension names, dimensions, units, scaling, and semantics before mapping them into internal fields.

Do not assume variance is standard deviation. Apply only the documented conversion needed by a particular method. Do not treat a missing quality extension as a passed quality check.

### 8.3 Metadata and contamination flags

The NASA challenge suggests showing metadata such as contamination flags. The Explorer SHOULD surface available quality/contamination information in a concise user-facing panel, with full technical detail accessible on demand.

Where supplied, preserve:

- pixel flags and source/background masks;
- contamination and nearby-source indicators;
- variance/uncertainty planes;
- PSF references and calibration status;
- saturation, bad-pixel, artifact, or other release-specific indicators;
- missing/unknown flag state.

Do not invent a generic “clean” status when a relevant flag plane is absent. The data layer must expose `NOT_AVAILABLE`/`UNKNOWN` where appropriate.

### 8.4 FITS validation

Before accepting a product for display or science, validate the applicable:

- readability and file integrity;
- actual HDU/extension layout;
- array dimensions and data type;
- image units and scaling;
- WCS and coordinate metadata;
- spectral wavelength/channel metadata;
- observation-time fields and time conventions;
- flags, variance, background, and PSF data;
- release/processing identity;
- checksum where available.

An unfamiliar or malformed header must not be silently normalized into plausible scientific data. Preserve unparsed metadata and flag fields that affect interpretation for review.

### 8.5 Display derivatives

The backend may produce PNG, JPEG, WebP, tiled images, or another suitable format for the browser. Each derivative must record:

- source product and parent cutout;
- selected channel/wavelength;
- coordinate region/FOV;
- output dimensions and pixel scale;
- stretch, normalization, color mapping, and scaling parameters;
- masking/quality behavior;
- resampling/reprojection behavior;
- software and processing version;
- generation timestamp and checksum where computed.

A display stretch changes visual appearance, not the underlying scientific measurement. The UI must identify display processing where relevant. Scientific measurements must use original data or a validated intermediate, never arbitrary browser-rendered pixel values.

### 8.6 Controlled artifact delivery

- Keep downloaded products and generated artifacts outside the public frontend static directory.
- Deliver them via controlled API routes or short-lived/signed references where appropriate.
- Validate artifact ownership and visibility before serving.
- Avoid accepting arbitrary client-supplied filesystem paths.
- Apply content-type, file-size, request-rate, and access controls.
- Cache public artifacts only when the cache key and access scope are safe.

---

## 9. Detector-Array Combination, Coaddition, and Signal-to-Noise

This section translates the NASA optional suggestion about the six detector arrays into a scientifically constrained capability.

### 9.1 Optional, not default

Detector-array combination is a `MAY` capability. The initial Explorer must work with separate, validated frames even if no combined product is available.

### 9.2 Array-aware grouping

If the selected product exposes detector-array identity:

- retain the original array ID for every source product and derived frame;
- group products only according to documented instrument/product semantics;
- do not assume six arrays can be summed into one undifferentiated image;
- distinguish array identity from broad band, spectral channel, and observation epoch;
- keep original per-array products accessible.

### 9.3 Conditions for coaddition

A coadd/stack may be generated only when the selected product documentation and validated method establish compatibility in the fields required for the operation, including:

- detector array and wavelength/channel;
- observation time/epoch grouping;
- calibration/release/processing state;
- unit and photometric scale;
- WCS, image geometry, pixel scale, and registration;
- PSF and background behavior;
- masks, flags, variance, and exposure information;
- weighting and uncertainty-propagation method.

Use a documented combination algorithm, such as an appropriate variance-aware weighting method, only after it has been scientifically selected and tested. The exact method and thresholds belong in `SCIENTIFIC_METHODOLOGY.md`.

### 9.4 Preserve time-domain discovery

Coadding data from different observation epochs can reduce noise for static sources but can smear moving objects, suppress variability, or erase time-dependent evidence.

Therefore:

- the default time-domain viewer MUST retain and display the original epoch frames;
- a coadded product MUST be a separate derived layer, not a replacement for the frame sequence;
- do not coadd across distinct epochs before the user has had access to the original temporal information;
- if a static-source stack is offered, label its epoch range and intended use;
- if an object may move or vary, evaluate whether stacking is inappropriate;
- retain the full input list, algorithm, parameters, weights, masks, uncertainty treatment, and output provenance.

A visually smoother or brighter image is not proof of improved scientific signal-to-noise for every object or use case.

---

## 10. Spectral Channel Selection and On-Demand Spectra

### 10.1 Channel selection

The Explorer SHOULD provide a spectral-channel or wavelength selector when the requested product contains supported channel information. Selection must be based on validated spectral metadata, not a guessed mapping from integer index to wavelength.

Represent the spectral dimension separately from observation time. If no compatible spectral plane is available for a requested epoch/channel, report the gap rather than substituting another wavelength without disclosure.

### 10.2 On-demand spectrum generation

The NASA challenge suggests generating spectra on the fly. HELIOS-X MAY support spectrum extraction when the product, source position/region, photometric method, calibration, PSF, and uncertainty fields support it.

The service must define and preserve:

- source identifier or user-selected coordinate/region;
- observation/product IDs and release;
- extraction method (for example, a supported forced-photometry or aperture method);
- source morphology/PSF treatment where relevant;
- aperture/footprint and background treatment;
- channel/wavelength values and units;
- flux values and units;
- statistical/systematic uncertainties where available;
- flags, masked/invalid channels, and missing data;
- calibration and processing version;
- method/software version and parameters.

A spectrum must not be presented as a blind source-discovery result if it was extracted at a user-supplied position. The IRSA Spectrophotometry Tool uses forced photometry at supplied source positions; its output is not a general blind detector.

### 10.3 Epoch handling in spectra

Do not silently combine spectra from distinct epochs, releases, or detector states as if they were one simultaneous spectrum. If time-resolved spectra are compared, expose the actual epoch per spectrum and apply a validated compatibility method.

Do not interpolate missing spectral channels without an explicit, justified, clearly labelled method. Missing channels must remain visible.

---

## 11. Image Blinking, Addition, Subtraction, and Temporal Analysis

### 11.1 Basic visual inspection

The core viewer may show two or more valid frames in sequence or blink between selected frames. This visual experience does not, by itself, establish astrometric alignment or physical motion.

A blink comparison may be made available for unregistered images only with a clear indication that the frames are not aligned and that apparent displacement may be caused by image geometry or WCS differences.

### 11.2 Frame compatibility before comparison

Before scientific comparison, check the applicable:

- distinct and trustworthy acquisition timestamps;
- same sky region and valid WCS;
- coordinate frame/reference epoch;
- compatible dimensions, orientation, pixel scale, and projection or a validated reprojection;
- same or explicitly compatible spectral channel/wavelength;
- compatible units, calibration, release, and processing state;
- PSF/background differences;
- masks, flags, variance, contamination, and missing pixels;
- registration residuals and propagated uncertainty;
- method-specific tolerances and assumptions.

Two images can be useful for visual browsing without being suitable for numerical subtraction or astrometric motion estimation.

### 11.3 Image subtraction / difference products

Image subtraction is optional and may be enabled only when the inputs are compatible and the scientific method is validated.

The pipeline must document and preserve:

1. source frame IDs and product references;
2. actual observation epochs and selected wavelengths/channels;
3. image registration method and quality/residuals;
4. photometric/flux scaling and background treatment;
5. PSF matching or reason it is not required;
6. masks, variance, and uncertainty propagation;
7. resampling/reprojection and its effects;
8. algorithm and software versions;
9. output artifact, warnings, and interpretation limits.

A subtraction result must not be generated from mismatched wavelengths or incompatible calibration just to create a visually compelling image. If prerequisites fail, return the appropriate unsupported, insufficient-data, or blocked outcome.

### 11.4 Image addition / stacking

Image addition is not arbitrary pixel-wise addition. It requires a defined scientific purpose and validated compatibility in units, calibration, time, WCS, registration, PSF, exposure, masks, background, weighting, and uncertainty handling.

The output must link to all input frames and clearly state whether it is a within-epoch combination, detector-array-specific combination, or cross-epoch stack. A cross-epoch stack may be inappropriate for moving or variable sources.

### 11.5 Apparent change versus physical motion

The data layer may report that a frame sequence exists or that visible images differ. A physical motion result requires more:

- actual distinct observation epochs;
- compatible observations;
- valid coordinate transforms/WCS;
- acceptable registration quality;
- source association across epochs;
- measured positions and uncertainties;
- a validated motion estimator and significance method;
- alternative explanations considered as required by the methodology.

Do not label a source as moving solely because it appears at different pixels in unregistered frames. Do not label brightness differences as variability without compatible photometry and uncertainty analysis.

The system must distinguish:

1. **Visual difference:** the displayed frames look different.
2. **Measured change:** a deterministic method reports positional or photometric change with uncertainty.
3. **Scientific interpretation:** a researcher/agent interprets validated measurements and limitations.
4. **Candidate prioritization:** a policy makes a candidate eligible for human attention.
5. **Scientific confirmation:** an expert conclusion requiring appropriate independent evidence; never an automatic agent status.

---

## 12. NASA Challenge Test Targets

The five targets below are required challenge test cases. Their inclusion is a test obligation, not a guarantee that IRSA currently has enough compatible SPHEREx observations to support every requested analysis.

| Target | Challenge context | Required HELIOS-X validation |
|---|---|---|
| Barnard's Star | Nearby star with prominent proper motion | Resolve the target identity, query actual SPHEREx coverage, inspect available epochs, and validate whether positional-change analysis is supported |
| Epsilon Indi B | Brown-dwarf pair associated with a nearby stellar system | Resolve the correct component/system identity; verify source confusion, field coverage, and whether the available data support image/spectral inspection |
| WISE 0855 | Fast-moving brown dwarf | Use the coordinates supplied in the challenge as the initial coordinate test; query actual coverage and test the motion-oriented inspection path |
| RW Aurigae | Variable-star test target | Verify actual observations, compatible photometric information, and whether variability analysis is supported |
| R Draconis | Variable-star test target | Verify actual observations, compatible photometric information, and whether variability analysis is supported |

### 12.1 WISE 0855 challenge-supplied coordinates

The NASA challenge statement supplies:

- **RA:** `08h 55m 10.831s`
- **Dec:** `−07° 14′ 42.53″`

These are the challenge-provided starting coordinates. Preserve the original sexagesimal input and convert through a validated coordinate library such as Astropy. Do not silently change the coordinate epoch/frame. If an authoritative catalogue provides updated astrometry, record it separately with source, epoch, and uncertainty rather than overwriting the challenge input.

### 12.2 Coordinate resolution for the other targets

The supplied challenge statement names Barnard's Star, Epsilon Indi B, RW Aurigae, and R Draconis but does not provide coordinates for all of them in the prompt text captured for this specification.

Do not invent or hard-code coordinates. Resolve each through a verified astronomical object identifier/catalogue (for example, SIMBAD or a reviewed authoritative catalogue), verify aliases and component identity, store the resolver/source and reference epoch, and include the resolved values in the target test fixture with citations and provenance.

For Epsilon Indi B specifically, disambiguate the intended system/component(s) before creating test expectations. Do not assume a blended image position is the position of each brown-dwarf component.

### 12.3 Required target test procedure

For every target:

1. resolve the target identifier and retain the coordinate source;
2. query the current public SPHEREx archive using a bounded FOV;
3. record query parameters, data-release inventory, returned observation IDs, and actual acquisition timestamps;
4. verify whether there are distinct epochs and whether products are compatible;
5. inspect channel/wavelength and detector-array availability;
6. validate FITS structure, WCS, units, flags, variance, PSF, and contamination metadata as applicable;
7. retrieve cutouts/previews and confirm that the target region is correctly centered;
8. verify that playback orders actual observations and displays gaps honestly;
9. run only scientific analysis methods whose prerequisites pass;
10. report unavailable data, ambiguous identity, insufficient time baseline, or unsupported methods explicitly.

### 12.4 Target result semantics

A target test passes if HELIOS-X correctly handles the target query and truthfully reports the data and capability state. It does not require a positive moving-object or variability detection.

Examples:

- If only one valid observation is returned, the Explorer should display it and state that multi-epoch comparison is unavailable for that query.
- If no records match the query, record a successful empty query rather than claiming the object does not exist.
- If the archive is unavailable, record a provider failure rather than a no-match.
- If data exist but required metadata/quality fail, prevent the dependent analysis while retaining the available image for appropriately labelled visual inspection where safe.

---

## 13. Auxiliary Catalogue and Mission Overlays

### 13.1 Gaia overlay

Gaia is an ESA mission and is an optional auxiliary source suggested by the NASA challenge. It can provide astrometry (including proper motion and parallax), photometry, and additional source information for many objects.

If implemented:

- pin and record the actual Gaia data release/product used;
- query through the Gaia Archive's documented interface;
- retain Gaia source IDs, coordinates, astrometric epoch, proper motion, parallax, uncertainties/covariances, and units where available;
- propagate positions to a relevant epoch only with a documented and suitable astrometric method;
- expose the catalogue layer separately from SPHEREx pixels;
- identify when Gaia coverage, magnitude limits, source class, crowding, or astrometric completeness make a match unsuitable;
- distinguish no match, ambiguous match, unavailable query, and failed query.

Do not assume Gaia contains every brown dwarf, faint infrared source, galaxy, or Solar System object. Gaia's absence of a counterpart is not proof of novelty.

### 13.2 WISE/NEOWISE overlay

WISE/NEOWISE is an optional auxiliary infrared survey source. It can supply image data, catalogue information, and time-resolved information across its own mission phases and observation epochs.

If implemented:

- use the IRSA WISE services and release documentation;
- pin the WISE/NEOWISE release or product family;
- preserve the mission phase, band, observation time range, image/catalogue ID, units, and quality metadata;
- expose WISE/NEOWISE layers separately from SPHEREx frames;
- retain wavelength differences and mission epoch differences in the UI and data contract;
- do not blend WISE measurements into SPHEREx photometry without a validated cross-instrument method;
- do not label a WISE/NEOWISE time series as SPHEREx temporal coverage.

### 13.3 Catalogue cross-match versus visual overlay

A visual overlay is a contextual display layer. A scientific cross-match is a separately validated operation with a documented catalogue release, coordinate epoch handling, search radius, matching method, uncertainties, ambiguous-match behavior, and provenance.

Do not treat drawing a catalogue marker near a SPHEREx source as an accepted match. The matching service must return its own explicit result and uncertainty/quality state.

### 13.4 Aladin Lite role

Aladin Lite may be used for contextual sky navigation, standard sky-survey layers, and catalogue/footprint overlays where its data and capabilities fit the use case. The custom SPHEREx temporal viewer remains responsible for displaying the selected SPHEREx frames and their observation-time sequence.

If used, pin a tested version, follow the project's licence/terms, preserve required attribution, and test how custom frame artifacts and coordinate systems are integrated. Do not assume an Aladin background survey is a SPHEREx image.

### 13.5 MOST and Horizons role

MOST and JPL Horizons are relevant for known Solar System objects with valid identifiers/ephemerides. They can support predicted-position queries and known-object checks. They do not provide a known orbit for hypothetical Planet X and must not be represented as blind unknown-object discovery tools.

---

## 14. Data Adapter Contracts and Normalized Metadata

### 14.1 Adapter result contract

Each adapter returns a typed result independent of the provider's raw response format. The normalized contract should include:

- source and service identifiers;
- product family and release;
- query ID and exact query parameters;
- query start/completion times;
- query status and record count;
- normalized product/observation records;
- warnings and structured errors;
- archive documentation reference;
- raw response metadata reference where retention is appropriate.

A conceptual record may contain these keys; this is not the native IRSA schema:

```json
{
  "source_name": "SPHEREx_IRSA",
  "product_family": "spectral_image",
  "release": "QR3",
  "query_id": "application-generated-id",
  "query_parameters": {
    "ra_deg": "validated numeric value",
    "dec_deg": "validated numeric value",
    "fov": "validated region definition"
  },
  "retrieved_at": "application-assigned UTC timestamp",
  "query_status": "SUCCEEDED",
  "record_count": 0,
  "records": [],
  "warnings": [],
  "errors": [],
  "documentation_reference": "official product documentation URL"
}
```

All values above are illustrative. Real product IDs, timestamps, coordinates, record counts, file references, and measurements must originate from the archive or validated application operations. Do not use illustrative placeholders in production records.

### 14.2 Product record fields

Preserve, where available:

- archive product ID and filename;
- durable archive reference and download URL;
- release and processing version;
- observation ID and actual acquisition time metadata;
- coordinate frame, WCS, and reference epoch;
- image dimensions and data units;
- detector/array identity;
- channel/wavelength and band metadata;
- calibration state and relevant gain/response references;
- flags, contamination, masks, and variance references;
- PSF metadata/reference;
- parent/cutout lineage;
- checksum and local controlled-storage reference;
- rights/citation/acknowledgement information where applicable.

### 14.3 Original metadata preservation

Normalize stable fields for application use, but retain or reference original headers and provider metadata. Unknown fields that could affect scientific interpretation must not be silently discarded. Preserve unparsed metadata or open a review item.

### 14.4 Query and product identity

Every query result must be traceable to:

- user query and normalized region;
- archive/interface and version where known;
- release/product family;
- exact applied filters;
- request and retrieval timestamps;
- returned IDs and result count;
- warnings/errors;
- local derivations and processing versions.

A later query over the same coordinates may return different products because the archive has changed. It must be represented as a new retrieval result rather than silently replacing the previous result.

---

## 15. Retrieval, Validation, and Evidence Workflow

The `SpherexIRSAAdapter` and associated data services should follow this sequence:

1. Validate RA/Dec, coordinate frame/epoch, FOV, filters, and configured resource limits.
2. Resolve a requested target name if the user supplied a name rather than coordinates.
3. Determine requested product family and preferred release from current capability configuration.
4. Check current official documentation and live archive availability.
5. Execute the metadata query and record the exact parameters and interface.
6. Distinguish successful empty results from query failures.
7. Validate each returned product's identity, release, observation metadata, and geometry.
8. Retrieve selected files or cutouts using documented access paths.
9. Validate file integrity, FITS structure, WCS, units, timestamps, spectral metadata, and quality/uncertainty fields required for the requested operation.
10. Generate browser-display derivatives if needed, preserving original files and complete derivation metadata.
11. Build the frame manifest from actual observations and actual acquisition timestamps.
12. Persist product references, query provenance, validation outcomes, checksums where feasible, and artifact references.
13. Return a truthful result to the Explorer or Workflow Orchestrator.
14. Permit downstream scientific operations only when their own capability and quality prerequisites pass.

The data adapter does not decide that an image contains a planet, a new object, or a confirmed motion signal.

---

## 16. Capability Matrix

Capabilities must be evaluated at runtime against the selected product, release, query, metadata, method implementation, and quality state. A general capability statement does not guarantee that every target/region has the same coverage.

### 16.1 Core Explorer capabilities — P0

| Capability | P0 expectation | Conditions |
|---|---|---|
| RA/Dec input | MUST | Explicit format, units, frame, and validation |
| FOV selection | MUST | Within documented server limits |
| SPHEREx observation discovery | MUST | Live archive or clearly labelled fixture during development; live support required for live-data claims |
| Actual observation-time metadata | MUST | Preserve time field/scale and acquisition semantics |
| Image/cutout display | MUST | Validated frame and controlled browser artifact |
| Chronological frame manifest | MUST | Based on actual timestamps, not channels or filenames |
| Video-like playback | MUST | Frontend controls frames supplied by the backend |
| Individual frame navigation | MUST | Every frame remains inspectable with metadata |
| Empty/failed/unavailable states | MUST | Distinct outcomes |
| Contamination/quality metadata | MUST where product supplies it | Missing fields remain unknown/unavailable |
| Coordinate URL sharing | SHOULD / challenge enhancement | Validate URL parameters and do not leak private data |

### 16.2 Optional and conditional capabilities — P1

| Capability | Status | Required validation |
|---|---|---|
| 102-channel selection/visualization | MAY | Product channel metadata, wavelength map, compatible frame semantics |
| Detector-array-specific coaddition | MAY | Array mapping, calibration, WCS/PSF/units/variance compatibility |
| On-demand spectrum extraction | MAY | Supported extraction method, supplied position/region, calibration, PSF, uncertainty |
| Image addition/stacking | MAY | Documented compatibility, alignment, scaling, weighting, masks, uncertainty |
| Image subtraction/difference | MAY | Distinct valid epochs, same compatible channel, registration and photometric normalization |
| Gaia overlay | MAY | Public release, astrometric epoch/uncertainty, catalogue service integration |
| WISE/NEOWISE overlay | MAY | Release/time/band metadata and independent overlay handling |
| Aladin Lite sky map | MAY | Integration fit, pinned version, data-layer semantics, attribution |
| Object-name resolution | SHOULD | Verified object IDs, alias resolution, coordinate reference/epoch |
| Source Discovery Tool integration | MAY | Method/interface availability and independent validation |
| MOST known-object workflow | MAY | Known object with usable ephemeris; not a blind discovery tool |

### 16.3 Not supported by SPHEREx data alone without further evidence

- treating 102 channels as 102 epochs;
- assuming every target has repeated usable observations;
- assuming the challenge's six-month survey statement means uniform local cadence;
- confirming a planet from an unmatched catalogue result;
- deriving a reliable orbit from one image or insufficient positions/baseline/precision;
- proving physical motion from unregistered pixel displacement;
- proving variability from incompatible flux measurements;
- treating a MOST image at a predicted location as independent confirmation;
- treating absent Gaia/WISE counterpart as evidence of a new object;
- merging WISE/NEOWISE and SPHEREx fluxes without a cross-instrument method;
- presenting detector-array coaddition as a default if calibration/compatibility is unknown;
- silently filling missing observations, epochs, channels, or measurements.

---

## 17. Error and Status Semantics

The exact public response envelope is defined by `API_SPECIFICATION.md`; persistence values by `DATABASE_SCHEMA.md`; canonical task statuses by `SYSTEM_ARCHITECTURE.md` and `INVESTIGATION_WORKFLOW.md`.

### 17.1 Query/retrieval outcomes

- `SUCCEEDED`: query/retrieval/validation completed according to its contract. This does not mean a scientific detection occurred.
- `SUCCEEDED_EMPTY`: an application-level query outcome indicating the query succeeded and returned zero matching records. If the canonical task model uses `SUCCEEDED` plus `record_count = 0`, preserve this distinction in the payload.
- `FAILED`: query/provider/network/parsing/integrity operation failed after bounded retries or due to a non-recoverable error.
- `RETRYING`: a transient error is being retried under the configured policy.
- `NOT_SUPPORTED`: requested archive interface/product/operation is unsupported.
- `INSUFFICIENT_DATA`: records exist but do not contain enough observations/metadata/coverage for the requested operation.
- `BLOCKED`: prerequisite or mandatory validation gate prevents the requested operation.
- `CANCELLED`: request was cancelled.
- `QUEUED` / `RUNNING`: request has not completed.

If `SUCCEEDED_EMPTY` is not a canonical task status, do not add it as a replacement enum without reconciling the canonical contracts. Use task status `SUCCEEDED` with an explicit zero-result outcome in the result payload.

### 17.2 Quality outcomes

Use the canonical outcomes:

- `PASSED`
- `FAILED`
- `INCONCLUSIVE`
- `NOT_AVAILABLE`
- `NOT_APPLICABLE`

Do not convert missing quality/variance/contamination data into `PASSED`.

### 17.3 Important distinctions

- Query failure is not a zero-result query.
- Zero-result query is not proof that an object does not exist.
- Missing metadata is not a default value.
- Unavailable data is not a physical non-detection.
- Invalid image registration is not physical motion.
- A frame sequence is not a completed scientific investigation.

---

## 18. Caching, Performance, and Resource Management

### 18.1 Cacheable records and artifacts

Potential cache entries include:

- metadata query results;
- archive product metadata;
- original retrieved products where local retention is justified;
- cutouts and browser previews;
- validated registration products;
- validated difference/stacking products;
- auxiliary catalogue query results.

### 18.2 Cache-key requirements

A cache key must include every parameter that changes the result, including relevant:

- archive source/interface;
- product family and release;
- product/observation identity;
- coordinate center/frame/epoch;
- FOV/geometry;
- selected channel/wavelength and detector array;
- image dimensions/pixel scale;
- stretch/normalization for a display preview;
- registration/comparison method and parameters;
- software/configuration version.

Never return a cached cutout generated for a different coordinate, FOV, channel, release, or processing version.

### 18.3 Cache freshness and release changes

Archive availability and data releases can change. Each cache entry must carry creation time, source release, and relevant metadata. Use documented freshness rules. Invalidate or version data when the archive publishes updated products. Never silently relabel an old QR2 artifact as QR3.

### 18.4 Bounded interactive requests

Use configuration-backed limits for:

- FOV/region area;
- maximum metadata results;
- maximum frames per interactive timeline;
- channel count per request;
- artifact/download size;
- simultaneous archive requests;
- cutout generation duration;
- preview memory/CPU use;
- public request rate.

Limits must be selected from actual archive behavior and deployment capacity, not guessed as scientific thresholds. The Explorer should return a clear message when a query exceeds an operational bound and offer supported narrower queries/pagination when appropriate.

### 18.5 Large/bulk processing

Bulk processing should use documented archive/cloud workflows or bounded background jobs. Do not keep a browser request open while downloading large products or running long analyses. The Explorer's simple frame query must not create an agentic investigation execution by default.

---

## 19. Security, Public Access, Licensing, and Attribution

### 19.1 Public Explorer access

The public Explorer may provide anonymous, read-only access to explicitly designated public SPHEREx records and public display artifacts. This does not make all backend resources public.

- Apply query rate limits and request-size limits.
- Restrict FOV, frame count, transfer size, and concurrent retrieval.
- Serve artifacts only through controlled paths.
- Do not expose raw local filesystem paths or private service credentials.
- Separate public archive products from private investigations and admin records.
- Require authentication/authorization for private investigation, review, user-console, and admin operations as required by `SECURITY.md`.
- Use backend authorization, never frontend-only hiding.

### 19.2 External data and prompt injection

Treat FITS headers, archive metadata, filenames, catalogue values, external pages, and retrieved text as untrusted data. They must never become agent instructions or override system policy. Agents can consume normalized, authorized evidence only.

### 19.3 Data handling and retention

- Do not commit downloaded survey data, large FITS products, private credentials, or generated runtime cache files to source control.
- Store raw products, previews, and derivative artifacts in controlled storage with explicit retention and cleanup rules.
- Preserve source IDs, retrieval timestamps, release, and checksum where feasible.
- Do not overwrite original data or previous scientific results.
- Keep the database for structured metadata and references; do not store large image arrays in normal relational rows.

### 19.4 Citation and acknowledgement

The current IRSA Quick Release documentation lists release-specific citation DOIs. The prior release page lists QR2 DOI `10.26131/IRSA652`; current QR3 documentation lists QR3 DOI `10.26131/IRSA662`. Re-check the current official citation instructions before every scientific publication or public report that requires formal acknowledgement.

Use the DOI corresponding to the actual release used. Retain citation metadata in each investigation and include the appropriate acknowledgement in exported scientific reports or publications where applicable.

The standard IRSA acknowledgement text is available on the official SPHEREx Quick Release page. Do not imply NASA, JPL, Caltech, IRSA, ESA, or the SPHEREx team endorses HELIOS-X, its candidate interpretations, or a Planet X claim.

### 19.5 Aladin terms

If Aladin Lite is embedded, preserve its required attribution/logo/link and follow the current integration terms. Pin a tested version and re-check the terms when upgrading.

---

## 20. Testing and Acceptance Criteria

Tests must validate both the public Explorer data path and the advanced investigation data path. A test plan is not proof that implementation has passed.

### 20.1 Adapter tests

- Valid query returns a typed result.
- Query parameters and applied filters are recorded accurately.
- Successful zero-result query differs from failed query.
- Product IDs, release, observation IDs, timestamps, WCS, and source references are preserved.
- Unsupported filters are not falsely represented as archive-applied filters.
- Missing/malformed metadata are explicit.
- Invalid or corrupt FITS products are rejected with a useful reason.
- Cutouts preserve required extension structure, WCS, and parent-product lineage.
- Download integrity/checksum validation works where configured.
- Transient errors trigger only bounded retries.
- Permanent errors do not retry indefinitely.
- Fixture adapter follows the same typed contract as the live adapter.

### 20.2 Release/product tests

- QR3 is preferred when available and appropriate.
- QR2 use is explicit and justified.
- QR1 is not selected as a current default.
- The adapter reads the actual release-specific product documentation.
- Actual extension names/dimensions/units are validated, not assumed.
- Release-specific PSF/calibration changes are covered where relevant.
- A changed release creates a distinguishable retrieval/analysis record.
- Roadmap products are not marked available until verified in IRSA.

### 20.3 Explorer frame and timeline tests

- RA/Dec input accepts only documented formats and units.
- FOV limits are enforced and reported clearly.
- URL coordinate/FOV parameters are parsed and validated.
- Observation discovery returns only actual archive records.
- The frame manifest preserves actual acquisition time separately from retrieval time and release date.
- Frames are sorted by actual validated acquisition time.
- Spectral channel changes do not advance the time axis.
- No fake frames are generated for missing epochs.
- Missing/ambiguous times are handled honestly.
- The viewer can inspect one frame even when multi-epoch data are unavailable.
- Playback and blinking work using actual frame artifacts.
- Failed or invalid frames are not silently rendered as real observations.
- Image artifacts are protected from arbitrary path access.

### 20.4 NASA target tests

For Barnard's Star, Epsilon Indi B, WISE 0855, RW Aurigae, and R Draconis:

- identity/alias resolution is correct and traceable;
- the coordinate source and reference epoch are stored;
- WISE 0855 accepts the challenge-supplied coordinates;
- actual archive coverage is queried rather than assumed;
- returned epochs and product releases are recorded;
- image cutouts are centered and WCS-validated;
- selected frames and channels are kept as separate dimensions;
- insufficient data and unavailable products remain explicit;
- the workflow never assumes a positive motion or variability result;
- results and limitations can be reproduced from the recorded query and product references.

### 20.5 Scientific capability tests

- Spectral channels cannot be used as time epochs.
- Time-domain analysis requires distinct, valid acquisition timestamps.
- Image comparison is blocked when required WCS/registration prerequisites fail.
- Visible shift is not labelled physical motion until the method's acceptance criteria pass.
- Variability is not claimed from incompatible photometry.
- Missing variance/quality/contamination information is handled according to the method.
- Difference images require compatible channels, calibration, alignment, and uncertainty handling.
- Coaddition preserves per-array identity and input lineage.
- Cross-epoch stacking does not replace original epoch frames.
- Gaia/WISE overlay records remain separate from SPHEREx pixel data.
- A Gaia/WISE no-match is not proof of a new object.
- MOST is not interpreted as a blind unknown-object detection.
- No automated result declares Planet X confirmed.

### 20.6 Auxiliary-source tests

If Gaia is enabled:

- the exact Gaia release is recorded;
- source IDs, epoch, proper motion, parallax, units, and uncertainties are preserved where supplied;
- epoch propagation is tested and method-versioned;
- ambiguous/no-match/provider failure states are distinct.

If WISE/NEOWISE is enabled:

- mission phase and release are recorded;
- image/catalogue/time-series products are kept distinct;
- actual WISE observation times are preserved;
- WISE data are not presented as SPHEREx frames;
- any cross-instrument photometric comparison is separately validated.

If Aladin Lite is enabled:

- supported overlays and coordinate interoperability are tested;
- version and attribution requirements are satisfied;
- Aladin survey imagery is never mislabeled as SPHEREx data.

### 20.7 Reproducibility tests

- Exact archive query and product release can be reconstructed.
- Retrieval timestamps, product IDs, and artifact references are persisted.
- Checksums are verified where feasible.
- Preview/cutout derivation parameters and software versions are recorded.
- The report links quantitative claims to source data and method outputs.
- Re-analysis preserves earlier results and identifies changed inputs/releases.

### 20.8 Acceptance gates

**Core Explorer data-source acceptance:**

- At least one documented live SPHEREx product and access path are verified end-to-end.
- A region query can discover actual products and observation times.
- A valid cutout/frame can be retrieved and displayed with metadata.
- A chronological frame manifest can be produced from actual acquisition timestamps.
- Empty query, failed query, unavailable product, malformed file, and missing metadata states are distinct.
- Data provenance is persisted.
- The viewer's basic time sequence is based on observation epochs, not spectral channels.

**Advanced scientific-data acceptance:**

- At least one selected analysis is demonstrated to be supported by the returned products.
- Registration/photometry/spectrum/comparison methods pass documented scientific tests.
- Uncertainty, quality flags, and limitations are visible.
- No unsupported physical-motion or discovery claim is made.

**Auxiliary/optional acceptance:**

- Each enabled Gaia, WISE, Aladin, array-coaddition, spectrum, or image-arithmetic feature has its own source, method, validation, and failure tests.
- Optional features remain disabled or visibly unavailable until those tests pass.

---

## 21. Implementation Order for the Data Source Layer

Implement the data-source layer in dependency order. The Explorer's core path is prioritized before optional analysis.

### Stage 0 — Official source and challenge review

1. Review the official NASA Space Apps challenge page and its Resources tab.
2. Review the current SPHEREx IRSA mission page, QR documentation, explanatory supplement, and archive user guide.
3. Record current release state and product access routes.
4. Verify exact product fields, extensions, units, WCS, observation timestamps, quality flags, variance, PSF, and spectral metadata from a representative file.
5. Record unresolved questions and source review date.

**Exit:** required source references are current and the selected product's semantics are understood well enough to design a fixture and adapter contract.

### Stage 1 — Coordinate/FOV and metadata-only discovery

1. Define typed sky-region query models.
2. Implement RA/Dec parsing and canonicalization.
3. Implement bounded FOV geometry and URL query validation.
4. Select one supported IRSA programmatic route.
5. Query metadata without downloading all matching images.
6. Record exact query, product/release, coverage, and observation times.

**Exit:** metadata discovery works against a live supported query or clearly labelled fixture and distinguishes empty results from failure.

### Stage 2 — One product and cutout validation

1. Retrieve one small representative spectral image/cutout.
2. Validate actual HDUs, WCS, units, channel metadata, flags, variance, PSF, and timestamps required by the initial viewer.
3. Preserve parent-product and release lineage.
4. Generate one controlled browser preview without modifying the source product.
5. Verify that the preview region corresponds to the requested coordinate/FOV.

**Exit:** one real archive product can be retrieved, validated, and represented by a traceable display artifact.

### Stage 3 — Frame manifest and time sequence

1. Discover more than one observation only where the archive actually provides them.
2. Build a versioned frame manifest.
3. Sort by validated acquisition time.
4. Keep channel and detector identity separate from epoch.
5. Expose frame metadata and coverage gaps.
6. Test viewer delivery for one-frame and multi-frame cases.

**Exit:** the Explorer can show one or more actual frames and does not invent observations or timeline events.

### Stage 4 — Five NASA target validation suite

1. Resolve target identities and coordinate references.
2. Use the challenge-supplied WISE 0855 coordinates.
3. Query actual archive availability for all five targets.
4. Save the query and product fixtures/manifest where licensing and size permit.
5. Record unavailable/insufficient data cases honestly.
6. Validate visual inspection, frame ordering, and metadata presentation.

**Exit:** every target has a reproducible test record, regardless of whether it yields multiple usable epochs.

### Stage 5 — First validated scientific comparison

1. Select a region/target with compatible repeated observations.
2. Verify units, WCS, release, calibration, channel, flags, variance, PSF, and timestamps.
3. Implement a documented registration/compatibility process.
4. Implement one supported comparison/measurement method.
5. Add known-answer synthetic tests and acceptance tolerances.
6. Preserve result and provenance separately from preview artifacts.

**Exit:** one scientific comparison is reproducible and validated. Do not claim more analysis than the selected input supports.

### Stage 6 — Explorer-to-investigation handoff

1. Define a snapshot of selected coordinate/FOV, observation IDs, product IDs, frame IDs, release, and analysis options.
2. Explicitly request an investigation; do not create one merely because the user browsed images.
3. Revalidate all references and capability prerequisites before execution.
4. Pass the snapshot into the Investigation Service/Workflow Orchestrator.
5. Preserve the source query and selection in investigation provenance.

**Exit:** the Agentic Engine can investigate a selected Explorer region without relying on mutable browser state.

### Stage 7 — Optional NASA enhancements

Add optional capabilities one by one only when the relevant data/method requirements are met:

1. spectral-channel browsing and on-demand spectrum extraction;
2. detector-array-specific coaddition/stacking;
3. image addition and subtraction;
4. Gaia overlay and validated cross-match;
5. WISE/NEOWISE overlay and time-series context;
6. Aladin Lite sky context/overlay;
7. expanded catalogue/source-discovery/MOST workflows.

**Exit:** each optional feature passes its own acceptance tests and does not weaken the core Explorer or scientific integrity rules.

### Stage 8 — Release and operational review

- verify current IRSA release/product documentation;
- verify attribution/citation and data-use conditions;
- test public endpoint limits and artifact access;
- review caching and stale-release behavior;
- verify security, logs, data retention, and cleanup;
- reconcile this document with API, database, backend, frontend, methodology, and testing specifications.

**Exit:** the implementation accurately distinguishes supported, unsupported, unavailable, unverified, fixture, and live archive capabilities.

---

## 22. Open Decisions and Required Evidence Before Production Claims

The following decisions must be finalized from current documentation or implementation tests rather than guessed:

1. Which specific QR3/QR2 product collection and programmatic query path will be the primary live integration?
2. Which archive fields are currently returned for observation time, time scale, channel/wavelength, detector array, and quality metadata?
3. Which cutout route preserves the extensions and WCS needed by the first supported workflow?
4. Does the chosen target region have enough distinct SPHEREx observation epochs for the intended temporal comparison?
5. Which exact source-name resolver/catalogue will provide the four targets without challenge-supplied coordinates?
6. Which product/method supports source detection and association for the selected target class?
7. Which detector-array mapping and coaddition rules are documented for the exact product release?
8. Which channel/wavelength selection will be used in the first demonstration?
9. Which image registration, photometric scaling, and difference-image method will be validated?
10. Which Gaia release and query service will be pinned if Gaia overlays are implemented?
11. Which WISE/NEOWISE release(s) and image/catalogue services will be used if WISE overlays are implemented?
12. Will Aladin Lite be used for contextual map navigation, and how will its frame overlays and attribution integrate with the custom viewer?
13. What FOV, request-size, frame-count, and rate limits are sustainable for the intended public deployment?
14. Which data-retention policy applies to downloaded products and generated cutouts/previews?

Each decision must have a recorded value, source, verification date, rationale, and affected contract. Unresolved decisions remain open and must not be represented as implemented behavior.

---

## 23. Cross-Document Synchronization Requirements

This specification has the following downstream dependencies:

| Document | Must reflect from this specification |
|---|---|
| `PRD.md` | Public Explorer requirements, NASA target tests, optional feature priorities, Explorer-to-investigation handoff |
| `SYSTEM_ARCHITECTURE.md` | IRSA adapter boundary, frame manifest, image artifact service, data/epoch/channel/array separation |
| `SCIENTIFIC_METHODOLOGY.md` | Compatibility rules, registration, coaddition, subtraction, motion/variability prerequisites, uncertainty |
| `INVESTIGATION_WORKFLOW.md` | Explicit snapshot/handoff and runtime capability gates |
| `API_SPECIFICATION.md` | Sky query, FOV, observation list, frame manifest, artifact delivery, target lookup, overlay and comparison contracts |
| `DATABASE_SCHEMA.md` | Queries, products, observations, frames, artifacts, channels, detector arrays, retrieval lineage, derived image products |
| `BACKEND.md` | Adapter implementation, cutout/preview, cache, frame, artifact delivery, optional science services |
| `FRONTEND.md` | Coordinate entry, FOV, frame timeline, playback, metadata, overlays, URL query state |
| `UI_UX_SPECIFICATION.md` | Public Explorer workflow and distinct optional controls/states |
| `TESTING_AND_ACCEPTANCE.md` | Five NASA target tests, real frame timeline, empty/unavailable/failure cases, science validation |
| `IMPLEMENTATION_PLAN.md` | Metadata-first retrieval, core Explorer path, target validation, science/agentic integration, optional enhancements |
| `MASTER_DEVELOPMENT_PROMPT.md` | Updated review order, source revalidation, implementation gates, and no-fabrication rules |

This document does not silently change their exact schema or endpoint definitions. Any contract update must be made in the authoritative document and then reflected in dependent documents.

---

## 24. References

The following references are grouped by their operational role. Verify each source's latest content before implementation or publication; archive products, endpoints, and competition resources may change.

### A. Challenge requirement

1. NASA Space Apps Challenge 2026 — Planet X and SPHEREx: <https://www.spaceappschallenge.org/2026/challenges/planet-x-and-spherex/>
2. NASA Space Apps Challenge Project Submission Guide: <https://www.spaceappschallenge.org/resources/project-submission-guide/>
3. NASA Space Apps Challenge 2026: <https://www.spaceappschallenge.org/2026/>

### B. SPHEREx primary archive and official product documentation

4. IRSA SPHEREx Mission Page: <https://irsa.ipac.caltech.edu/Missions/spherex.html>
5. IRSA SPHEREx Quick Release Overview: <https://irsa.ipac.caltech.edu/data/SPHEREx/docs/overview_qr.html>
6. SPHEREx Archive User Guide: <https://caltech-ipac.github.io/spherex-archive-documentation/>
7. SPHEREx Data Products — IRSA Archive Documentation: <https://caltech-ipac.github.io/spherex-archive-documentation/spherex-data-products/>
8. SPHEREx Explanatory Supplement: <https://irsa.ipac.caltech.edu/data/SPHEREx/docs/>
9. IRSA Program Interfaces: <https://irsa.ipac.caltech.edu/voapi.html>
10. IRSA SIA Interface: <https://irsa.ipac.caltech.edu/ibe/sia.html>
11. IRSA Astroquery Documentation: <https://astroquery.readthedocs.io/en/latest/ipac/irsa/irsa.html>
12. PyVO Documentation: <https://pyvo.readthedocs.io/>
13. IRSA Cloud Data Access: <https://irsa.ipac.caltech.edu/cloud_access/>
14. AWS Open Data Registry — SPHEREx Quick Release: <https://registry.opendata.aws/spherex-qr/>

### C. SPHEREx tools and tutorials

15. SPHEREx Data Explorer: <https://irsa.ipac.caltech.edu/applications/spherex/>
16. SPHEREx Cutout Tool: <https://irsa.ipac.caltech.edu/data/SPHEREx/docs/cutout_tool.html>
17. SPHEREx Spectrophotometry Tool: <https://irsa.ipac.caltech.edu/onlinehelp/spherex/spherex/sp.html>
18. SPHEREx Source Discovery Tool tutorial: <https://caltech-ipac.github.io/irsa-tutorials/spherex-source-discovery-tool-demo/>
19. SPHEREx MOST: <https://irsa.ipac.caltech.edu/applications/MOST/>
20. SPHEREx MOST Instructions: <https://irsa.ipac.caltech.edu/applications/MOST/instructions.html>
21. SPHEREx Intro Tutorial: <https://caltech-ipac.github.io/irsa-tutorials/spherex-intro/>

### D. Mission and instrument context

22. NASA SPHEREx Mission: <https://science.nasa.gov/mission/spherex/>
23. NASA SPHEREx Universe Map/Open Science: <https://science.nasa.gov/open-science/spherex-universe-map/>
24. NASA SPHEREx by the Numbers: <https://science.nasa.gov/blogs/spherex/2025/03/11/nasas-spherex-by-the-numbers/>
25. JPL SPHEREx Mission: <https://www.jpl.nasa.gov/missions/spherex/>
26. SPHEREx Instrument: <https://spherex.caltech.edu/page/instrument>
27. SPHEREx Data Products: <https://spherex.caltech.edu/page/data-products>
28. SPHEREx Science Products: <https://spherex.caltech.edu/page/science-products>

### E. Astronomy software and visualization

29. Astropy Documentation: <https://docs.astropy.org/en/stable/>
30. Astropy Coordinates: <https://docs.astropy.org/en/stable/coordinates/>
31. Astropy Time: <https://docs.astropy.org/en/stable/time/>
32. Astropy FITS: <https://docs.astropy.org/en/stable/io/fits/>
33. Astropy WCS: <https://docs.astropy.org/en/stable/wcs/>
34. Aladin Lite Documentation: <https://aladin.cds.unistra.fr/AladinLite/doc/>
35. Aladin Lite API Documentation: <https://cds-astro.github.io/aladin-lite/>
36. Astroquery Documentation: <https://astroquery.readthedocs.io/>

### F. Auxiliary catalogues and services

37. Gaia ESA Archive: <https://gea.esac.esa.int/archive/>
38. Gaia DR3 Documentation: <https://gea.esac.esa.int/archive/documentation/GDR3/>
39. IRSA WISE/NEOWISE Mission Page: <https://irsa.ipac.caltech.edu/Missions/wise.html>
40. IRSA WISE Image Service: <https://irsa.ipac.caltech.edu/onlinehelp/wise/wise/overview.html>
41. SIMBAD: <https://simbad.cds.unistra.fr/simbad/>
42. VizieR: <https://vizier.cds.unistra.fr/>
43. JPL Horizons: <https://ssd.jpl.nasa.gov/horizons/>

### G. Scientific and responsible-use references

44. NASA — Hypothetical Planet X / Planet Nine: <https://science.nasa.gov/solar-system/planet-x/>
45. Akeson et al., SPHEREx image/spectrophotometry processing pipeline: <https://arxiv.org/abs/2511.15823>
46. A Pan-STARRS1 Search for Planet Nine: <https://doi.org/10.3847/1538-3881/ad24e9>
47. SPHEREx spectra for nearby brown dwarfs: <https://arxiv.org/abs/2607.00543>

---

## Review Note

**Last reviewed:** 2026-11-01.  
**Review scope:** NASA Space Apps challenge requirements supplied for HELIOS-X; current IRSA/SPHEREx product documentation and access references; Astropy and Aladin tooling references; Gaia and WISE auxiliary-source references; alignment with the updated HELIOS-X Product Requirements Document, System Architecture, and Backend Engineering Specification.

Product availability, release versions, endpoint definitions, catalogue releases, access policies, and challenge Resources-tab contents can change. Re-check official sources before each implementation milestone and scientific release. Do not claim live data support, target coverage, detector-array combination, spectral extraction, catalogue overlay, or scientific detection until the relevant capability has been implemented and independently tested.

**End of Data Source Specification.**
