# HELIOS-X UI/UX Specification

**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Document:** `UI_UX_SPECIFICATION.md`  
**Version:** 2.0 (MVP — architecture-aligned)  
**Status:** Design and implementation specification  
**Frontend stack:** React, TypeScript, Vite, Tailwind CSS  
**Backend API:** FastAPI, versioned REST API  
**Product surfaces:** Public Explorer, User Console, Admin Console  
**Theme:** Dark-first, with a deliberately designed light mode  
**Design direction:** Graphite Observatory — warm charcoal, restrained observatory amber, and clear scientific data presentation

---

## 1. Purpose

This document defines the user experience, navigation, visual design system, layouts, core screens, interaction behavior, responsive behavior, and frontend acceptance criteria for HELIOS-X.

HELIOS-X has two connected product capabilities:

1. **Public SPHEREx Sky Image Explorer:** a public-facing, read-only experience for searching a sky region by supported coordinates, inspecting actual observations, viewing image frames over time, and using basic frame playback/navigation.
2. **Agentic Scientific Investigation Engine:** an explicitly initiated scientific workflow for data validation, deterministic analysis, evidence/provenance, candidate assessment, reports, and human review where implemented.

The product is delivered through three separately buildable frontend applications:

- `frontend/` — Public Explorer.
- `user-console/` — authenticated investigation workspace.
- `admin-console/` — restricted administrative/operational interface, limited to approved and implemented functions.

The Public Explorer and the investigation workspace must remain connected without becoming the same experience. A visitor must be able to browse public sky data without creating an account, investigation, or agent run. An explicit handoff is required to take a selected Explorer context into the investigation workflow.

This specification owns **UX behavior and visual design**. It does not own the canonical repository structure, implementation phases, API schemas, database schema, or operational procedures. Those remain in their respective reference documents. This document does not claim that any screen, endpoint, scientific method, or integration has already been implemented or tested.

## 2. Product UX Goals

The interface must:

1. Make public sky exploration easy to understand and use.
2. Make starting and continuing an investigation straightforward.
3. Show what the system is doing, what has completed, and what needs attention.
4. Connect measurements and candidate assessments to their underlying evidence and methods.
5. Show uncertainty, data-quality issues, observation coverage, and missing-data limitations clearly.
6. Make failures recoverable where recovery is supported.
7. Look distinctive, polished, minimal, and appropriate for scientific exploration.
8. Remain usable on desktop, tablet, and mobile screens.
9. Use real API state and data—not decorative charts, fake progress, fabricated observations, or placeholder interactions presented as real.
10. Avoid overwhelming users with internal implementation details unless they choose to inspect them.
11. Preserve a clear boundary between public read-only exploration, private investigation work, and restricted administration.
12. Keep the MVP focused on a complete Explorer-to-investigation handoff and a complete investigation-to-report journey, without making either journey depend on unimplemented features.

### 2.1 UX principles

- **Two connected capabilities:** Sky exploration and scientific investigation are distinct product experiences with an explicit handoff.
- **Evidence over spectacle:** visual effects must never obscure scientific content.
- **Progress must be real:** progress reflects backend state; unknown progress remains unknown.
- **Traceable results:** important results link to source data, tasks, methods, and evidence.
- **Uncertainty is part of the result:** never hide quality flags, gaps in coverage, missing data, or contradictory evidence.
- **Safe by default:** prevent accidental duplicate runs and make cancellation behavior clear.
- **Progressive disclosure:** show useful summaries first, with deeper technical details available on demand.
- **Visual restraint:** use strong hierarchy, spacing, typography, and alignment rather than decorative gradients, excessive cards, or glow effects.
- **Consistent semantics:** the same status uses the same label, meaning, and semantic treatment across all three applications.
- **Backend is authoritative:** the UI never invents scientific or persistent application state.
- **Accessible by design:** keyboard access, readable contrast, clear focus, and alternatives to color-only or drag-only interactions are required.

## 3. Product Surfaces and MVP Scope

### 3.1 Surface A — Public SPHEREx Sky Image Explorer (`frontend/`)

This is the primary public-facing NASA challenge experience.

The Explorer should allow a visitor to:

- Enter supported RA/Dec coordinates and a supported field of view (FOV).
- Submit a sky-region query and inspect the returned observation coverage.
- View available image frames for the selected region.
- Navigate frames in their actual observation-time order.
- Play, pause, and step through available frames.
- Use blink/comparison behavior only where the backend provides the required compatible frames and the feature is implemented.
- Inspect the selected frame's available metadata, quality information, and source/provenance references.
- See when no observations exist, a product is unavailable, or coverage is incomplete.
- Explicitly hand a selected region/observation context to the User Console for investigation, where that capability is implemented.

**Scientific interaction rules:**

- Temporal playback must use distinct actual observation timestamps.
- SPHEREx's 102 spectral channels are wavelength samples, not 102 time epochs. A spectral-channel selector, if exposed, must be a separate control from the temporal timeline.
- Do not fabricate frames, interpolate missing observations, or draw a continuous availability bar across periods without observations.
- A six-month all-sky survey cadence must not be presented as proof that the selected region has observations exactly every six months.
- The count and dates of available observations must come from the actual query result.
- The Explorer is read-only. Searching, viewing, playing, stepping, blinking, and changing display preferences must not create an investigation or start agent execution.
- The handoff to investigation must be an explicit action. The User Console must show the imported selection and let the user review it before the investigation is created or run.

The Explorer must not become a full observatory workstation in the MVP. Advanced WCS editing, arbitrary catalogue overlays, image processing, spectrum analysis, multi-source overlays, and file upload are not implied by this specification. Expose them only when the backend, scientific method, and implementation support them.

### 3.2 Surface B — User Console (`user-console/`)

The User Console is the authenticated, private investigation workspace. Preserve the original six core investigation screens:

1. **Workspace** — active/recent investigations, actions needing attention, and ways to continue work.
2. **New Investigation** — define the scientific question or target, validate data/capabilities, and configure supported analysis.
3. **Investigation Monitor** — track execution, tasks, logical agent roles, progress, errors, retries, and cancellation.
4. **Evidence & Candidates** — inspect observations, measurements, evidence, quality outcomes, candidate assessments, and limitations.
5. **Report Viewer** — read a scientific report, navigate evidence references, and view report version information.
6. **Settings & Capabilities** — inspect supported data sources and analysis capabilities, with minimal display preferences.

A user may start an investigation directly in the User Console or from an explicit Explorer handoff. Private investigations and their related resources must be loaded from the backend and scoped to the authenticated user's authorization.

### 3.3 Surface C — Admin Console (`admin-console/`)

The Admin Console is a separate, restricted interface. It is not a public dashboard and is not required to be a feature-rich MVP deliverable.

Its UX must:

- Show only operational functions that have implemented, approved backend support.
- Keep administrative actions separate from ordinary investigation workflows.
- Display service health/capability state only when it comes from real backend checks.
- Provide audit/history views only when corresponding backend records and authorization exist.
- Require clear confirmation for supported sensitive operations.
- Explain the outcome of an administrative action and provide a trace/reference where available.
- Never imply that a function exists merely because a button or navigation item is visible.

Do not add a database browser, arbitrary SQL console, shell/terminal, raw filesystem browser, secret viewer, unrestricted provider proxy, arbitrary agent-prompt editor, or tool-permission editor. The Admin Console must not bypass scientific quality gates, alter measurements, silently rewrite evidence, or change a candidate's scientific meaning.

### 3.4 Features included only if implemented end to end

These must not appear as enabled UI features until the backend, persistence, permissions, and required tests support them:

- Human review submission and review history.
- PDF export or other downloadable formats.
- Live push updates using WebSockets or server-sent events.
- Advanced visual analysis tools such as image overlays, interactive WCS viewers, or spectrum comparison plots.
- User accounts, team permissions, and collaboration beyond the implemented authenticated-user boundary.
- Advanced filtering, saved searches, and bulk operations.
- Administrative configuration changes or operational actions.
- Public URL sharing or permalink restoration of Explorer state.
- Object-name resolution, catalogue overlays, or external catalogue lookup.

If a capability is planned but unavailable, either omit it or mark it clearly as unavailable with a useful explanation. Do not present a non-functional control as if it works.

### 3.5 Explicitly out of scope for the MVP

- A decorative dashboard full of unrelated KPI cards.
- A simulated live mission-control display disconnected from real tasks.
- Fake observations, fake frames, or charts used to fill empty space.
- A complex drag-and-drop workflow builder.
- Configuring arbitrary agent prompts or tools from the UI.
- Presenting candidate records as confirmed discoveries.
- Large-scale collaboration or enterprise administration unless required by a confirmed deployment.
- A full observatory image-processing workstation unless those capabilities are implemented and validated.
- Automatic agent execution caused by passive Explorer browsing or playback.
- Artificially smoothing observation timelines by inventing missing dates.

## 4. Information Architecture and Navigation

### 4.1 Application separation

The three frontends are separate applications with different access boundaries and primary tasks. They may share design tokens and carefully chosen UI components when that is supported by the repository architecture, but they must not be forced into one oversized navigation shell.

- **Public Explorer:** public, read-only discovery and image inspection.
- **User Console:** authenticated, private investigation work.
- **Admin Console:** restricted operational actions and views.

Do not expose User Console private-resource navigation or Admin Console functions in the anonymous Explorer.

### 4.2 Public Explorer navigation and routes

The Explorer should prioritize the sky image and the query workflow.

| Route | Purpose |
|---|---|
| `/explore` | Main coordinate query, sky image viewer, observation list/timeline, and frame inspection |

A query-state URL or shareable permalink may be added only if it is defined by the API/security contract and does not expose private data. Do not make URL sharing a prerequisite for the MVP.

The Explorer header may include the HELIOS-X wordmark, “Sky Explorer” label, compact help/about entry, and a link to the User Console only if the deployment provides that destination. Avoid account/profile controls on the anonymous Explorer unless a real authenticated integration exists.

### 4.3 User Console navigation and routes

Use a restrained, persistent application shell.

**Desktop primary navigation:**

- Workspace
- Investigations (only if a dedicated list route is implemented; otherwise Workspace is the investigation list)
- Capabilities
- Settings

The investigation detail area provides contextual tabs or sub-navigation:

- Overview / Monitor
- Evidence & Candidates
- Reports

**Proposed User Console routes:**

| Route | Purpose |
|---|---|
| `/workspace` | Active work and recent investigations |
| `/investigations` | Investigation list, if implemented separately from Workspace |
| `/investigations/new` | New investigation flow |
| `/investigations/:investigationId` | Investigation overview and monitor |
| `/investigations/:investigationId/evidence` | Evidence and candidate exploration |
| `/investigations/:investigationId/reports` | Reports for an investigation |
| `/reports/:reportId` | Read a specific report |
| `/capabilities` | Supported data and analysis capabilities |
| `/settings` | Theme and basic preferences |

The global shell may include the HELIOS-X wordmark, current workspace label, theme switch, and a compact account indicator only if the authentication implementation provides it. Do not add notification bells, profile menus, global search, or collaboration controls unless they have real behavior.

### 4.4 Admin Console navigation and routes

The Admin Console should use a restrained shell with navigation generated only from implemented administrative capabilities. Potential categories such as operational overview, audit history, or approved configuration are not commitments to implement those features.

Do not publish route names for administrative actions until the API and authorization contract define them. Never make an admin route accessible by hiding it only in the frontend.

### 4.5 Navigation behavior

- Preserve the current investigation context when moving between its monitor, evidence, and reports.
- Use breadcrumbs for deep investigation/report pages where they help orientation.
- Browser back/forward navigation must work.
- Refreshing an investigation page must reload its state from the backend, not from temporary in-memory assumptions alone.
- Unsaved form changes should trigger a clear warning before navigation when practical.
- Do not put transient UI state into permanent URLs unless it is useful and supported for sharing/restoring the view.
- The Explorer-to-User-Console handoff must carry only a validated, supported selection context. The User Console must revalidate the selection on the backend.
- A failed handoff must not create a partial or misleading investigation.

## 5. Visual Design System

### 5.1 Design direction — Graphite Observatory

The interface should feel like a premium scientific instrument: calm, minimal, precise, warm, and visually memorable.

Use:

- Deep warm graphite/charcoal as the primary dark environment.
- Warm ivory and clean neutral surfaces for light mode.
- Restrained observatory amber/brass for primary actions and selected emphasis.
- Clear neutral typography and fine, purposeful borders.
- Semantic colors only for actual success, warning, error, and information states.

Avoid:

- Purple/blue as the primary brand palette.
- Blue-purple gradients and neon gradients.
- Excessive cyan or green highlights.
- Glow around every card, button, graph, or navigation item.
- Decorative glass effects that reduce text or chart clarity.
- Heavy shadows, oversized pill-shaped cards, and nested card stacks.
- Large decorative hero areas that push the actual Explorer or investigation content below the fold.

The main visual richness should come from layout proportion, typography, image quality, precise spacing, subtle surface separation, and restrained accent use—not from visual effects.

### 5.2 Dark-mode palette

| Token | Hex | Usage |
|---|---|---|
| `color.bg` | `#10110F` | Main application background |
| `color.surface` | `#181A16` | Main panels and cards |
| `color.surface-elevated` | `#20221D` | Menus, dialogs, drawers, elevated sections |
| `color.surface-hover` | `#292B25` | Hovered rows and controls |
| `color.border` | `#36382F` | Subtle panel separators; decorative boundaries only |
| `color.border-strong` | `#6C6D60` | Meaningful control boundaries and selected outlines |
| `color.text-primary` | `#F4F1E8` | Main text and key values |
| `color.text-secondary` | `#C4C0B4` | Supporting text |
| `color.text-muted` | `#A6A193` | Muted labels where contrast remains sufficient |
| `color.primary` | `#D9A85F` | Primary action fill and restrained selected emphasis |
| `color.primary-hover` | `#E4B975` | Primary hover state |
| `color.primary-text` | `#241A0D` | Text/icons on primary amber fill |
| `color.link` | `#E8BD78` | Inline links and keyboard focus emphasis |
| `color.success` | `#9BC7A9` | Success/healthy state |
| `color.warning` | `#E8BD78` | Warning state |
| `color.error` | `#F08A80` | Error/failed state |
| `color.info` | `#B1BDB7` | Informational state |
| `color.neutral-status` | `#C4C0B4` | Queued, inactive, or unknown state |

### 5.3 Light-mode palette

| Token | Hex | Usage |
|---|---|---|
| `color.bg` | `#F4F1E9` | Main page background |
| `color.surface` | `#FFFFFF` | Cards and primary panels |
| `color.surface-elevated` | `#FAF8F2` | Menus and dialogs |
| `color.surface-hover` | `#F0EDE5` | Hovered rows and controls |
| `color.border` | `#D8D2C5` | Subtle dividers and panel separation |
| `color.border-strong` | `#8A8576` | Meaningful control boundaries and selected outlines |
| `color.text-primary` | `#25251F` | Main text |
| `color.text-secondary` | `#5D5B51` | Supporting text |
| `color.text-muted` | `#6D6A60` | Muted labels after contrast verification |
| `color.primary` | `#B47A37` | Primary action fill |
| `color.primary-hover` | `#BC8544` | Primary hover state |
| `color.primary-text` | `#23180B` | Text/icons on primary amber fill |
| `color.link` | `#80531E` | Inline links and keyboard focus emphasis |
| `color.success` | `#286447` | Success labels |
| `color.warning` | `#80531E` | Warning labels |
| `color.error` | `#A4312B` | Error labels |
| `color.info` | `#4F6664` | Informational labels |
| `color.neutral-status` | `#5D5B51` | Queued, inactive, or unknown states |

The light theme must be designed and tested deliberately; do not implement it by simply inverting dark-mode colors. Verify actual text and UI contrast in rendered components. The palette values are design tokens, not a substitute for accessibility testing.

### 5.4 Color usage rules

- Amber is the primary brand/action accent. Use it sparingly so important actions and selected states remain clear.
- Do not use amber for every chart series, warning, link, and selection simultaneously; the same color must not have conflicting meanings.
- Success, warning, error, information, and neutral colors are semantic, not decorative.
- Every status must include readable text; never rely on color alone.
- Use one consistent color for the same dataset/series across related charts.
- Scientific chart palettes must be selected for distinguishability, data meaning, and color-vision accessibility. They may use additional hues when necessary; the product brand palette must not constrain scientific data encoding.
- Avoid gradients for quantitative scientific values unless the visualization explicitly requires a sequential scale and includes a labelled legend.
- Do not place long body text in accent colors.
- Verify text contrast and meaningful UI boundary contrast in both themes. Use WCAG 2.2 AA as the target: normal text contrast 4.5:1, large text 3:1, and applicable non-text controls/graphics 3:1.
- Do not style candidate prioritization or preliminary evidence as a confirmed discovery.

### 5.5 Typography

Use a clean, widely available sans-serif font for interface text, such as **Inter** or a system-font fallback. Do not make the product dependent on a font that may not be installed or loaded.

Use a restrained type scale:

- Page title: 26–32 px desktop, 22–26 px mobile.
- Section heading: 18–22 px.
- Panel/card heading: 15–17 px.
- Body text: 14–16 px.
- Supporting labels: 12–14 px, with sufficient contrast.
- Scientific values, coordinates, timestamps, IDs, and tabular numeric data: use tabular numerals; use a readable monospace face only where fixed-width alignment improves verification.

Avoid very small text for metadata users need to verify scientific results. Long descriptions and reports should use comfortable line height and a readable content width.

### 5.6 Layout, spacing, borders, and elevation

- Use a consistent spacing scale based on 4 px or 8 px increments.
- Prefer open layouts and clear alignment over excessive card nesting.
- Use subtle borders to define panels and stronger boundaries for meaningful controls.
- Shadows should indicate elevation, not decorate every component.
- Use moderate corner radii consistently: typically 8 px for controls and 10–12 px for panels.
- Avoid huge rounded cards, oversized empty hero sections, and dashboard-style KPI grids.
- Use charts and tables only when they communicate real data or useful state.
- Use a stable grid and consistent content widths in the User Console; keep the Explorer viewer spacious and image-led.
- Avoid placing several nested panels inside each other. Use tabs, split panes, drawers, and progressive disclosure instead.

### 5.7 Motion and visual effects

- Use brief transitions, generally around 120–200 ms, for opening panels, changing tabs, and hover/focus feedback.
- Do not animate fake scanning, data arrival, object movement, or workflow progress.
- Frame playback must advance through actual available frames, not generated interpolations.
- Respect `prefers-reduced-motion`.
- Avoid perpetual pulsing/glowing elements.
- Animation must not delay access to controls or obscure changing information.
- Do not use parallax, decorative starfields, or animated orbital graphics behind scientific content.

## 6. Application Shells and Shared Components

### 6.1 Public Explorer shell

- Keep the header compact so the sky image remains the primary visual area.
- Place coordinate/FOV query controls in a clear top search region.
- Keep observation navigation and the timeline close to the image viewer.
- Place metadata in a collapsible side inspector or drawer.
- Do not use the User Console sidebar navigation in the Explorer.
- On smaller screens, move metadata and observation details into drawers or stacked panels without hiding frame controls.

### 6.2 User Console shell

The header should contain:

- HELIOS-X wordmark/product label.
- Current section or investigation context.
- Theme toggle.
- Optional authenticated-user indicator only if real authentication context is available.
- Optional compact service indicator only if it is tied to an actual backend health/capability endpoint.

On desktop, use a collapsible sidebar with labels and icons. A collapsed sidebar must have accessible names/tooltips. Selected navigation must be visually clear in both themes.

Main content requirements:

- Use a stable content grid and predictable page heading position.
- Keep the primary action visible near the relevant page title or workflow step.
- Allow long scientific tables or technical payloads to scroll within their own region rather than stretching the entire application.
- Preserve a visible page title and resource identity in all investigation detail screens.

### 6.3 Admin Console shell

- Use a separate restricted shell with only approved operational navigation.
- Do not reuse User Console routes as a shortcut to privileged operations.
- Make the active administrator context clear where supported.
- Provide clear confirmation and outcome feedback for sensitive actions.
- Do not expose raw secrets, database credentials, internal paths, arbitrary logs, or unrestricted infrastructure controls.

### 6.4 Shared components

Create reusable components where they genuinely improve consistency:

- Buttons and icon buttons.
- Text inputs, selects, coordinate fields, date/time controls, and checkboxes.
- Status badges with text labels.
- Alerts and inline validation messages.
- Tabs, breadcrumbs, and page section navigation.
- Progress indicators.
- Empty states and error panels.
- Paginated data tables.
- Evidence references.
- Measurement/value-with-unit displays.
- Confirmation dialogs.
- Copyable IDs and coordinates.
- Report section navigation.
- Tooltips for technical terms.
- Timeline/frame controls.
- Metadata definition lists and source-reference blocks.

The same state must use the same label, meaning, and semantic color treatment across all screens.

## 7. Screen Specifications

### 7.1 Public Explorer — Sky Query, Image Viewer, and Observation Timeline

**Purpose**

Let a visitor inspect a selected sky region across actual available SPHEREx observations with minimal friction and clear scientific context.

**Desktop layout**

Use an image-led workspace with three functional areas:

1. **Query/header area:** compact coordinate and FOV form, query action, and current query summary.
2. **Main viewer area:** the selected sky image/frame is the dominant visual element.
3. **Context/inspector area:** available observation list or frame metadata, shown in a collapsible panel where needed.

A bottom timeline/control strip should sit within the Explorer work area, not cover the scientific image or essential content.

Suggested desktop proportions are flexible, not fixed: the image should receive the largest share of the available space; the inspector should remain narrow enough to preserve image readability. Do not force all panels to remain open simultaneously on smaller screens.

**Query controls**

- RA and Dec fields must have persistent labels and explicit units/format hints.
- FOV must expose only supported values/ranges.
- Accept coordinate formats only when the backend/API supports them; explain the expected format.
- Validate ranges and malformed input before submission, then handle backend validation errors.
- Prevent duplicate query submission while a query is pending.
- Display the submitted query coordinates and FOV in the results context.
- Object-name resolution may be offered only if an approved backend capability provides it. It is not assumed by this specification.

**Image viewer**

- Keep the image centered and visually dominant.
- Provide only viewer controls that are implemented and backed by real image data (for example, fit/reset view or zoom, where supported).
- Display a clear loading state while the requested frame is being retrieved.
- Show a useful empty/error state if the image is missing, malformed, or unavailable.
- Do not stretch an image in a way that distorts its aspect ratio or coordinate interpretation.
- If WCS/coordinate-grid overlays are implemented, label them and ensure they correspond to the selected product's actual metadata.
- Do not imply that a normal preview is a calibrated scientific analysis product unless the product metadata supports that claim.

**Observation list and coverage**

Show only observation records returned by the backend. Each row/item may display:

- Actual observation timestamp, preferably with an explicit UTC label.
- Observation/product identifier where available.
- Relevant product/release information.
- Channel/filter label only when supplied by the API.
- Quality/availability summary where available.

The list should make it easy to select an actual observation. If there are no observations, show a successful empty-coverage state rather than a fabricated timeline.

**Timeline and playback**

- The timeline must be built from the actual frame manifest/observation timestamps.
- Show discrete frame markers or intervals based on returned data.
- Gaps in observation coverage must remain visible as gaps; do not draw a continuous line that implies unobserved data exists.
- Provide previous frame, next frame, play, pause, and current-frame display where implemented.
- Playback must move only through actual available frames in timestamp order.
- If the sequence contains one frame, disable playback and explain that there is no second temporal frame.
- If playback reaches the last available frame, follow the documented end behavior (stop or loop only if an explicit loop control exists).
- A blink/comparison mode may compare two real selected frames only when compatible images and the comparison behavior are supported.
- The temporal timeline and spectral channel selection must be visually and functionally separate. Never place the 102 spectral channels as 102 timeline epochs.

**Frame/observation inspector**

Show available metadata in a structured, scannable layout:

- Observation timestamp and time basis.
- Product/observation identifier.
- Data release or product version, where available.
- Spectral channel information, where applicable.
- Quality flags and known limitations.
- Coordinate/FOV context.
- Source/provenance references.
- Preview/download action only if the content endpoint and permission policy support it.

Use a compact summary first and a collapsible “Technical details” area for extended metadata. Missing fields must be labelled unavailable or omitted according to the API contract; do not invent values.

**Explorer-to-investigation handoff**

- Provide a clear action such as “Investigate this region” only when the handoff is implemented.
- Show what context will be transferred: coordinates, FOV, selected observation/frame references, time selection, and any supported options.
- Transfer only validated identifiers/parameters; do not pass raw client filesystem paths or arbitrary provider URLs.
- The User Console must revalidate the selection and show a review step.
- Browsing, playing, stepping, blinking, or changing the selected frame must never silently create an investigation or start an agent run.
- If the User Console is unavailable or the handoff fails, preserve the public Explorer state and show a recoverable message.

**Explorer states**

- **Initial:** concise instructions and coordinate/FOV input; no fake sky query results.
- **Query loading:** stable viewer layout with a clear loading indicator.
- **Invalid query:** field-level errors and retained user input.
- **Successful observations:** actual image, frame list, and timestamp-based timeline.
- **Successful empty result:** explain that no observations were returned for the requested region/constraints; do not imply that no astronomical object exists.
- **Partial coverage:** show available frames and visible temporal gaps.
- **Product unavailable/corrupt:** show a precise non-success state and keep other valid frames available where possible.
- **API/provider failure:** distinguish failed retrieval from a successful empty query; provide retry only when safe.
- **Handoff pending/succeeded/failed:** make the result clear without claiming investigation creation before backend confirmation.

### 7.2 User Console Screen 1 — Workspace


#### Purpose


Help users continue useful work rather than display static performance metrics.


#### Layout


**Top region**

- Page title: “Workspace” or “Your investigations”.

- Primary action: “New investigation”.

- Optional short explanatory line.


**Main region**

1. **Continue working:** active or recently updated investigations, ordered by meaningful activity or status.

2. **Needs attention:** failed runs, blocked tasks, incomplete data, or report/review actions that require attention.

3. **Recent investigations:** a compact searchable/filterable list if enough records exist.

4. **First-use state:** when no investigations exist, explain the first step and offer a create-investigation action.


Avoid a grid of generic cards showing totals such as “Total Agents”, “Total Sources”, or “Total Evidence” unless each number supports a genuine workflow decision.


#### Investigation row/card content


Show only available information:

- Investigation name or scientific question summary.

- Target label or coordinates, if relevant.

- Current status.

- Last updated time.

- Latest execution status where available.

- A meaningful next action, such as “Resume”, “View results”, or “Inspect failure”.


Do not invent a target name or use placeholder values as real records.


#### Interactions


- Selecting an item opens that investigation.

- “New investigation” opens the creation flow.

- “Resume” opens the existing execution/monitor state; it must not silently start a second run.

- Filters and sorting must affect the displayed API data, not just change a decorative label.

- If no records match a filter, show a filter-specific empty state with a way to clear filters.


#### States


- **Initial loading:** skeleton rows with stable layout.

- **Empty workspace:** explain how to start the first investigation.

- **Empty filtered results:** explain that no records match current filters.

- **API failure:** show a retry action and preserve useful context.

- **Populated workspace:** show real resources and current statuses.


---


### 7.3 User Console Screen 2 — New Investigation


#### Purpose


Help the user define a scientifically meaningful investigation using only capabilities the current deployment supports.


#### Suggested flow


Use a short multi-section form or compact stepper. Do not create a long wizard with unnecessary screens.


**Section A — Scientific question**

- Investigation name.

- Scientific question or objective.

- Optional notes.


**Section B — Target and search**

- Supported target type, such as sky position or another explicitly implemented search type.

- Required parameters for that target type.

- Clear units and coordinate conventions.

- Input validation near the relevant field.


**Section C — Data source and availability**

- Show data sources/products returned by the capabilities endpoint.

- Explain unavailable or unsupported data.

- Display relevant time/coverage constraints only when the selected source supports them.

- Provide a preflight result for the requested operation.


**Section D — Analysis options**

- Show only analysis methods supported by the selected data and current deployment.

- For disabled options, provide a short reason if showing them is useful.

- Do not imply that every one of the eight agent roles must run for every investigation.


**Review**

- Summarize target, selected source/product, analyses, and known limitations.

- Allow the user to return and edit fields.

- The final action should describe the actual action, for example “Create investigation” or “Create and run” only if that exact operation is implemented.


#### Validation behavior


- Validate required fields before submission.

- Validate coordinates and ranges where applicable.

- Prevent submission while required capability checks are unresolved.

- On server-side validation errors, map errors to fields where possible.

- Preserve entered values after recoverable errors.

- Prevent repeated submission while the request is pending.

- Show a successful creation state only after the backend confirms success.


#### Post-submission behavior


Navigate to the created investigation. If the workflow was started, show the execution ID and current status returned by the backend. If the investigation was only created, do not imply analysis has begun.


---


### 7.4 User Console Screen 3 — Investigation Monitor


#### Purpose


Make long-running scientific work understandable and auditable.


#### Layout


**Header**

- Investigation name and ID.

- Overall execution status.

- Created/updated timestamps where useful.

- Primary action according to state: “Start run”, “Cancel run”, “View results”, or “Retry task” when permitted.


**Progress region**

- Real execution status.

- Progress percentage only if the backend has enough information to calculate a meaningful value.

- Completed/total task counts when available.

- Clear distinction between cancellation requested and cancellation completed.


**Workflow region**

- A readable ordered task list or dependency graph.

- Each task shows task type, logical agent/component role, status, attempt number when relevant, and a concise result/error summary.

- Use a graph only if dependency structure is meaningful and remains readable. A well-designed task list is preferable to an overcomplicated graph.


**Details region**

- Selecting a task opens a detail panel with input/output summary, timestamps, method version when applicable, retry history, and error information.

- Show structured result summaries, not hidden chain-of-thought or private internal reasoning.


#### Logical agent roles


Where applicable, display these project roles:

1. Investigation Planner Agent

2. Data Retrieval Agent

3. Data Quality Agent

4. Change Detection Agent

5. Motion Analysis Agent

6. Spectral & Photometry Agent

7. Catalogue Matching Agent

8. Validation & Hypothesis Agent


Show only tasks actually created for the execution. An unused agent role must not be displayed as failed or running. The Workflow Orchestrator is a system component, not a ninth agent.


#### Status presentation


Use the project's canonical task statuses:

- `QUEUED`

- `RUNNING`

- `SUCCEEDED`

- `FAILED`

- `RETRYING`

- `CANCELLED`

- `BLOCKED`

- `NOT_SUPPORTED`

- `INSUFFICIENT_DATA`


Execution-level statuses may include `CANCEL_REQUESTED` and other aggregate statuses defined by the API. Display human-readable labels while retaining the canonical status in details/tooltips when useful.


#### Interactions


- Poll actual status using the API behavior defined by the API specification.

- Pause polling when the page is not visible or the execution is terminal, where practical.

- Do not assume a task failed because one poll request failed; distinguish a temporary status-fetch error from a backend task failure.

- Cancellation requires confirmation if it may discard in-progress work.

- Once cancellation is requested, clearly label it as pending until the backend confirms completion.

- Retry is available only when the API confirms the operation is permitted.

- Refreshing the page must restore the execution view from API data.


---


### 7.5 User Console Screen 4 — Evidence & Candidates


#### Purpose


Allow users to inspect what the system measured, how it was measured, whether it passed quality checks, and why a candidate is or is not prioritized.


#### Layout


Use sub-tabs or clearly separated sections:

- Observations / data products.

- Measurements.

- Evidence.

- Candidates and assessments.


Show only sections supported by the current investigation. Empty sections should explain why no records are available rather than implying an error.


#### Evidence list


Each evidence item should show, when available:

- Short title/type.

- Summary.

- Originating task.

- Source product or observation reference.

- Method name and version.

- Quality outcome.

- Timestamp.

- Known limitations.


Selecting evidence opens a detail view with structured metadata, supporting measurements, and references. Large technical JSON may be available in a collapsible “Technical details” section rather than shown by default.


#### Measurement display


A measurement should include:

- Quantity name.

- Value and unit.

- Uncertainty, when supplied by the method.

- Quality flags.

- Method/version.

- Input evidence references.


If uncertainty is unavailable, say “Uncertainty unavailable” or the equivalent supported label; do not display zero or make up a value.


#### Candidate details


Display:

- Provisional designation/label.

- Current candidate status.

- Assessment version and timestamp.

- Supporting evidence.

- Contradictory evidence.

- Alternative explanations.

- Missing data and limitations.

- Suggested next steps, when available.


Use canonical candidate statuses:

- `NEEDS_MORE_DATA`

- `NOT_PRIORITIZED`

- `PRIORITIZED_FOR_REVIEW`

- `BLOCKED_BY_QUALITY_ISSUE`


Never style “prioritized for review” as “confirmed”. There must be no implied confirmed-planet badge or celebratory discovery animation.


#### Visualization rules


- Use image, spectral, motion, or coordinate plots only when the selected observation type and supported method provide valid data.

- Include axis labels, units, legends, and uncertainty indicators where relevant.

- Provide a table or textual summary for important chart values.

- Distinguish measured data from modelled/derived data.

- If a visualization is unavailable, show the underlying metadata/results rather than a blank chart.

- Sample/demo data must be visibly labelled as such.


---


### 7.6 User Console Screen 5 — Report Viewer


#### Purpose


Present a readable, traceable scientific report without hiding evidence, uncertainty, or limitations.


#### Layout


**Header**

- Report title.

- Investigation reference.

- Report status/version.

- Generated timestamp.

- Available actions such as copy link or download only when implemented.


**Main content**

- Executive summary.

- Scope and data sources.

- Methods used.

- Findings and measurements.

- Quality outcomes and uncertainties.

- Candidate assessment, if produced.

- Alternative explanations and unresolved issues.

- Limitations and recommended next steps.

- Evidence/source references.


Use a left-hand contents rail on wide screens if the report is long. On narrow screens, use a compact section menu.


#### Evidence navigation


- Clicking an evidence reference opens the evidence details or navigates to the relevant evidence section.

- Links to unavailable or removed resources must show a useful error rather than silently doing nothing.

- A report must show its version and must not silently change under the same version identifier.


#### Generation states


- **Queued/running:** show that generation is in progress and provide a way to return or refresh status.

- **Succeeded:** show the generated content.

- **Failed:** show a meaningful error and a retry action only if retry is supported.

- **Incomplete:** clearly label any partial output and explain what is missing.

- **No report:** explain how to request generation.


#### Export


Show only output formats that are supported and tested in the current deployment. Markdown/HTML rendering may be used for the MVP; PDF should not be shown as an available action until it is implemented.


---


### 7.7 User Console Screen 6 — Settings & Capabilities


#### Capabilities


Show the sources, products, and analysis operations that the current deployment supports, based on the backend capabilities endpoint.


Each capability should communicate:

- Display name.

- Support state.

- Required input/data, when relevant.

- Limitations or constraints.

- Whether it is experimental, unsupported, or available.


Do not make broad mission/data-source claims based solely on a planned adapter.


#### Settings


For MVP, keep settings minimal:

- Theme: system, dark, or light.

- Optional accessibility/display preferences if implemented.


Avoid adding account management, team settings, notification preferences, or API key management unless the corresponding capability exists.


---



### 7.8 Admin Console — Operational Views

**Purpose**

Give authorized administrators a narrow, auditable view of implemented operational functions without turning the Admin Console into an unrestricted infrastructure console.

**Layout**

- Compact header with product label and restricted-administration context.
- Narrow navigation generated from actual implemented administrative capabilities.
- Main content focused on the selected operational task.
- Use tables or structured status lists where they help locate a service, configuration item, or audit event.
- Avoid vanity KPI cards, decorative uptime meters, or status indicators without a real backend source.

**Permitted presentation patterns, only when implemented**

- Service/application health and readiness summaries from backend checks.
- Data-source/capability availability as reported by the backend.
- Security/administrative audit history exposed through an authorized API.
- Approved configuration settings with validation, confirmation, and recorded outcome.

**Safety and interaction rules**

- Never display secret values, API keys, database credentials, or private key material.
- Do not expose raw SQL, shell, arbitrary filesystem paths, or unrestricted logs.
- Do not provide unrestricted access to user investigations or private reports merely because a visitor is an administrator.
- Destructive or sensitive actions require confirmation and a clear explanation of impact.
- Show success only after the backend confirms completion.
- Preserve an audit reference/record where the backend provides one.
- Do not allow administrative operations to bypass scientific gates, rewrite measurements, or silently alter evidence.
- If no administrative API function exists, omit the corresponding page/control.

---

## 8. Loading, Empty, Error, and Partial States


Every screen must define these states as applicable.


### 8.1 Loading


- Keep page layout stable with skeletons or a compact progress indicator.

- Avoid full-screen spinners for small requests.

- Do not show empty-state messaging while a request is still loading.

- Prevent duplicate mutations while a request is pending.

- Keep the origin of progress clear: API workflow progress versus UI loading.


### 8.2 Empty


An empty state should explain:

- What is currently missing.

- Whether that is normal.

- What the user can do next, if any action is available.


Examples:

- No investigations yet → create an investigation.

- No evidence generated → explain that the workflow has not produced evidence yet or did not support that analysis.

- No candidates → state that no candidate records were produced for this investigation; do not suggest that no astronomical objects exist.

- No reports → offer report generation when permitted.


### 8.3 Error


- State what failed in user-understandable terms.

- Offer a retry only when retry is safe and supported.

- Preserve user input where possible.

- Display a request ID or support identifier for unexpected server errors.

- Do not expose stack traces, secrets, internal filesystem paths, or raw provider credentials.


### 8.4 Partial and insufficient data


- Distinguish “no result”, “analysis not run”, “analysis unsupported”, “failed”, and “insufficient data”.

- A partial workflow must not be represented as fully successful.

- Preserve successful task results when another independent task fails.

- Reports must disclose missing data and incomplete analysis.

- Candidate/evidence presentation must retain quality outcomes and contradictory information.


---


## 9. Frontend State and API Integration


### 9.1 Source of truth


The backend is authoritative for:

- Investigation and execution IDs.

- Task and execution statuses.

- Progress data.

- Evidence and measurements.

- Candidate assessments.

- Report status and content.

- Supported capabilities.


The frontend may manage ephemeral UI state such as the open tab, expanded panel, form editing state, and current theme. It must not invent persistent scientific state.


### 9.2 Typed contracts


- Use TypeScript types generated from or checked against the backend OpenAPI contract where practical.

- Avoid duplicating request/response definitions by hand without tests.

- Validate and handle unknown enum values defensively.

- Never assume optional API fields are present.

- Treat `null`, omitted fields, and empty arrays according to the API schema.

- Keep API calls in a service/query layer, not embedded throughout presentational components.


### 9.3 Async and duplicate-request behavior


- Disable or guard submit buttons while a mutation is pending.

- Use backend idempotency for operations that support it; client-side button disabling is not enough.

- Ignore or safely handle stale responses when the user changes resource or leaves a screen.

- Abort obsolete read requests where practical.

- Poll using the documented API behavior and stop when an execution becomes terminal.

- Handle temporary polling errors separately from the execution’s actual status.

- After a mutation, refresh or update the relevant resource state using the confirmed API response.


### 9.4 Form state


- Show field-level validation errors where possible.

- Keep entered form values after recoverable network/server errors.

- Avoid submitting invalid or unsupported analysis options.

- Warn about unsaved changes when navigation could discard them.

- Do not show a success notification before the server confirms success.


### 9.5 Persistence


- Resource state must survive browser refresh by reloading from the API.

- If local storage is used, limit it to appropriate UI preferences or draft data and document its purpose.

- Do not store access tokens, secrets, or large scientific data in local storage without a reviewed security need.

- Theme preference may be stored locally if implemented safely.


---


### 10. Responsive Layout

Responsive behavior must be designed for each application rather than shrinking one desktop dashboard to fit every screen.

#### 10.1 Public Explorer — desktop

- Keep coordinate/FOV controls visible without dominating the image.
- Give the main sky frame the largest area.
- Keep the observation list/inspector collapsible.
- Keep the timeline and playback controls attached to the viewer.
- Do not allow metadata panels to obscure selected image features.

#### 10.2 Public Explorer — tablet

- Allow the metadata inspector to collapse into a drawer.
- Keep essential query and frame controls visible.
- Allow the image to use most of the available width.
- Keep observation selection accessible without forcing a multi-column desktop layout.

#### 10.3 Public Explorer — mobile

- Use one primary content column.
- Put coordinate/FOV controls in a compact form panel.
- Keep image aspect ratio and image controls clear.
- Present the observation list and detailed metadata in expandable sheets/drawers.
- Use a compact timeline with accessible previous/next controls; do not require precise dragging to select a frame.
- Playback controls must remain reachable and must not overlap the image.
- If a scientific table or timeline cannot remain readable at the available width, provide a usable list alternative.

#### 10.4 User Console — desktop

- Persistent sidebar with labels and icons.
- Multi-column layouts only where they aid comparison.
- Evidence/task details may appear in a side panel.
- Tables may use the available width, with deliberate horizontal scrolling for genuinely wide scientific data.

#### 10.5 User Console — tablet

- Sidebar may collapse.
- Multi-column panels should reflow into fewer columns.
- Keep critical workflow controls visible.
- Avoid popovers or detail panels that cover essential navigation without a clear close action.

#### 10.6 User Console — mobile

- Use one primary content column.
- Replace wide tables with responsive row cards or deliberate horizontal scrolling where tabular relationships matter.
- Keep actions reachable and touch targets comfortably sized.
- Charts must resize and retain visible labels/legends; if unreadable, provide a table alternative.
- Task dependency graphs should simplify to an ordered list.
- Forms should use full-width controls with clear labels.
- Dialogs should fit the viewport and keep their actions accessible.
- Do not require hover to discover important information.

#### 10.7 Admin Console — responsive behavior

The Admin Console may be desktop-oriented where operations require wide tables, but all essential actions and audit information must remain usable on tablet widths. Avoid placing critical controls outside the viewport. Mobile support must not be represented as complete until it is tested.

Responsive behavior should be verified at narrow, medium, and wide viewport sizes. Breakpoints should be chosen consistently in the implementation rather than scattered across components.

---

## 11. Accessibility and Usability


Use WCAG 2.2 AA as the target for relevant accessibility checks.


Minimum requirements:

- All interactive controls work with a keyboard.

- Focus is visible and follows a sensible order.
- Pointer targets should be at least 24 × 24 CSS pixels unless a WCAG 2.2 exception applies; provide larger touch targets where practical.
- Text can be resized to 200% without loss of content or functionality.

- Inputs have persistent labels; placeholder text is not the only label.

- Validation and error messages are associated with the relevant field.

- Color is not the only indicator of status.

- Main text and meaningful interactive controls meet contrast requirements.

- Icons have accessible names when they convey meaning.

- Decorative icons are hidden from assistive technology.

- Dialogs manage focus and return focus when closed.

- Progress updates are announced without overwhelming the screen reader.

- Charts have a text summary or accessible data alternative.

- Respect reduced-motion settings.

- Avoid flashing, continuous pulsing, and unnecessary motion.


---


## 12. UI Content and Scientific Language


Use clear, neutral, scientifically responsible language.


Preferred wording:

- “Candidate requires further review.”

- “No catalogue match was found in the searched catalogue.”

- “Insufficient observations for motion analysis.”

- “Uncertainty information is unavailable.”

- “Analysis is not supported for the selected data product.”

- “Cancellation requested” while shutdown is still in progress.


Avoid:

- “New planet discovered” without independent scientific verification.

- “No object exists” when there is no result.

- “100% certain” or unqualified confidence claims.

- “Analysis complete” when required tasks failed or were blocked.

- Fake precision or fabricated measurements.


Technical status values may be shown in detailed views, but the primary UI should use readable labels while preserving accurate semantics.


---


## 13. UI Testing and Acceptance Criteria

### 13.1 Shared component tests

Test:

- Status badges display the correct label and semantic color.
- Buttons show disabled/loading state and cannot trigger duplicate mutations.
- Forms associate validation messages with inputs.
- Dialogs can be opened, closed, and used with a keyboard.
- Evidence references navigate to the correct resource.
- Dark and light themes preserve hierarchy and readability.
- Focus indicators are visible in both themes.
- Important state is not communicated by color alone.

### 13.2 Public Explorer tests

Test:

- Coordinate and FOV validation, including server-side validation errors.
- Loading, invalid input, successful result, successful empty result, partial coverage, provider failure, and corrupt/missing product states.
- Observation rows use actual returned metadata and timestamps.
- Timeline ordering uses actual observation timestamps.
- Timeline gaps remain visible; no missing frames are fabricated or interpolated.
- Spectral channel selection is separate from temporal navigation.
- Previous/next/play/pause behavior uses only available frames.
- Playback is unavailable or safely disabled for a single-frame sequence.
- A temporary API polling/retrieval error is not misrepresented as a scientific observation failure.
- Explorer browsing and playback do not create investigations or agent runs.
- The explicit handoff passes only supported context and shows the correct outcome.
- The image viewer retains correct aspect ratio and usable controls at agreed viewport widths.

### 13.3 User Console screen-level tests

**Workspace**

- Empty, populated, loading, filtered-empty, and API-error states.
- Selecting an investigation opens the correct resource.
- “Resume” does not silently create a new execution.

**New Investigation**

- Required fields and range validation.
- Unsupported capabilities are not submitted.
- Failed submission preserves input.
- Navigation occurs only after successful creation.

**Investigation Monitor**

- Correct display for every documented status.
- Progress reflects API data and may be unknown.
- Temporary polling failure is not shown as a task failure.
- Cancellation request and completion are visibly distinct.
- Retry is unavailable when the API does not permit it.

**Evidence & Candidates**

- Measurements show units and uncertainty when available.
- Missing uncertainty is not displayed as zero.
- Supporting and contradictory evidence are distinguishable.
- Candidate states are never shown as confirmed discovery.

**Report Viewer**

- Pending, succeeded, failed, and incomplete states.
- Evidence links work.
- Report version and limitations are visible.
- Unsupported export formats are not offered.

**Settings & Capabilities**

- Display data from the actual capabilities endpoint.
- Unavailable capabilities are clearly labelled.

### 13.4 Admin Console tests

Only where the relevant features exist:

- Unauthorized users cannot access administrative routes or APIs.
- Hidden frontend navigation is not treated as an authorization control.
- Only approved backend operations appear.
- Sensitive actions require confirmation.
- Success/failure feedback matches the backend response.
- Audit history and administrative outcomes are traceable where implemented.
- Secrets and privileged configuration values are not exposed.

### 13.5 End-to-end acceptance

The Public Explorer journey must verify, using supported data or clearly labelled fixtures:

1. Open the Public Explorer.
2. Enter a supported RA/Dec and FOV.
3. Submit the query.
4. Inspect returned observations and coverage.
5. Open an available image frame.
6. Navigate or play through actual observation timestamps.
7. Confirm that coverage gaps are not filled with fabricated frames.
8. Explicitly hand the selected context to the User Console, if that integration is implemented.
9. Confirm the handoff is reviewed before investigation creation/run.

The User Console journey must verify:

1. Open Workspace.
2. Create an investigation directly or from a validated Explorer handoff.
3. View the created investigation.
4. Start a workflow.
5. See execution/task progress from the API.
6. Open available results/evidence.
7. Request report generation.
8. Open the completed report.
9. Verify that limitations and provenance are present.

Each journey must also verify at least one failure or insufficient-data path so the UI does not show false success.

### 13.6 Release gates for UI

The UI is acceptable for MVP when:

- All screens included in the release have defined states.
- Public Explorer and investigation workflows remain clearly separated.
- The primary Explorer-to-investigation handoff passes where implemented.
- The investigation-to-report journey passes.
- Core actions are connected to functioning API operations.
- No critical navigation or data-display bug remains.
- Responsive layouts work at agreed viewport sizes.
- Keyboard operation and contrast have been checked in both themes.
- Backend status and scientific results are never replaced by decorative placeholders.
- Unsupported features are omitted or clearly marked unavailable.
- The interface distinguishes failed, cancelled, blocked, unsupported, insufficient-data, and successful-empty outcomes.
- Known limitations are documented.

---

## 14. Frontend Feature Boundaries

The canonical repository structure and exact folder paths are owned by `IMPLEMENTATION_PLAN.md` and `FRONTEND.md`. This section defines UI feature ownership only; it is not a competing folder-structure specification.

### 14.1 Public Explorer feature ownership

- Coordinate/FOV query form and validation.
- Observation discovery and coverage state.
- Sky image/frame viewer.
- Actual-timestamp timeline and frame navigation.
- Playback/blink controls where supported.
- Observation metadata inspector.
- Explicit investigation handoff.
- Public Explorer loading, empty, error, partial, and unavailable states.

### 14.2 User Console feature ownership

- Workspace and investigation list.
- New investigation form.
- Investigation overview/monitor.
- Task and execution status.
- Evidence and measurement presentation.
- Candidate assessment presentation.
- Report viewer.
- Capabilities and minimal user preferences.

### 14.3 Admin Console feature ownership

Only approved, implemented operational functions. Administrative route and component ownership must follow the backend authorization contract. Do not create placeholder screens or privileged frontend-only behavior.

### 14.4 Shared UI ownership

Shared tokens/components may include:

- Design tokens and theme primitives.
- Buttons, inputs, tables, dialogs, tabs, tooltips.
- Status labels and semantic state treatment.
- Loading, empty, error, and confirmation patterns.
- Coordinate/time/value formatting components where semantics match.

Do not force surface-specific controls into a generic component when their behavior differs materially. The Explorer timeline, investigation task monitor, and Admin audit table are different interaction patterns even if they share visual primitives.

---

## 15. UI Design Implementation Guidance

This specification defines the design requirements, not the overall project implementation sequence. Follow `IMPLEMENTATION_PLAN.md` for the authoritative project build order and phase gates.

When implementing a UI feature:

1. Confirm the relevant UX requirements and screen states in this file.
2. Confirm the route, API contract, permissions, and available data with the owner documents and actual backend contract.
3. Build from the shared design tokens and accessible components.
4. Connect the control to a real supported API operation.
5. Implement loading, empty, error, partial, and unsupported states as applicable.
6. Add component/screen tests for the interaction and state transitions.
7. Verify desktop, tablet, and mobile behavior.
8. Review both themes, keyboard access, focus, contrast, and reduced-motion behavior.
9. Do not add visual polish that obscures scientific meaning or implies an unsupported feature.

Build only the screens and result visualizations required by the supported end-to-end product capabilities. Extend the UI as additional scientific capabilities become supported.

---

## 16. Decisions to Confirm During Implementation

Keep this list short and resolve each item only when required:

1. Exact plotting/image-viewer library, if the selected scientific workflow needs one.
2. Whether the Explorer requires an embedded sky-viewer library beyond the delivered image-frame viewer.
3. Whether the MVP needs a formal human-review action or only read-only candidate assessment.
4. Whether progress polling is sufficient for the first release.
5. Exact viewport sizes used by responsive acceptance tests.
6. Whether reports need PDF export in the MVP.
7. Final component and frontend testing libraries.

Do not delay the main workflows while adding optional visualizations or collaboration features.

---

## Appendix A. Design Token Summary

### Dark theme

- Background: `#10110F`
- Surface: `#181A16`
- Elevated surface: `#20221D`
- Border: `#36382F`
- Strong control boundary: `#6C6D60`
- Primary text: `#F4F1E8`
- Secondary text: `#C4C0B4`
- Primary action: `#D9A85F`
- Primary action text: `#241A0D`
- Link/focus: `#E8BD78`
- Success: `#9BC7A9`
- Warning: `#E8BD78`
- Error: `#F08A80`

### Light theme

- Background: `#F4F1E9`
- Surface: `#FFFFFF`
- Elevated surface: `#FAF8F2`
- Border: `#D8D2C5`
- Strong control boundary: `#8A8576`
- Primary text: `#25251F`
- Secondary text: `#5D5B51`
- Primary action: `#B47A37`
- Primary action text: `#23180B`
- Link/focus: `#80531E`
- Success: `#286447`
- Warning: `#80531E`
- Error: `#A4312B`

These are proposed design tokens. Verify actual rendered contrast, state distinction, and chart usability in the implementation before release.

## Appendix B. Design Research and Applied References

The following references inform the interaction patterns in this specification. They are design references, not dependencies or requirements to copy another product's full feature set.

1. **ESA ESASky** — a public astronomical data exploration interface that supports sky-region exploration and access to image observations, catalogue data, spectral/light-curve data, and other data services. HELIOS-X adopts the principle of region-centred exploration and context-specific result tools, not ESASky's full feature set.  
   https://sky.esa.int/

2. **Aladin Lite documentation** — demonstrates browser-based sky visualization with pan/zoom and the ability to visualize image surveys and overlay tabular/catalogue or footprint data. HELIOS-X may use an appropriate viewer library if justified by the implementation; this specification does not mandate Aladin Lite or unimplemented overlays.  
   https://aladin.cds.unistra.fr/AladinLite/doc/

3. **NASA Worldview** — demonstrates an image-led viewer with layer controls, date navigation, playback, keyboard shortcuts, and shareable view state. The applicable lesson for HELIOS-X is to keep image navigation and time controls close to the visualization and make them operable without relying solely on a mouse.  
   https://worldview.earthdata.nasa.gov/

4. **NASA research on sparse imagery availability in Worldview** — documents the need to represent missing temporal coverage as gaps and provide available-imagery dates rather than imply continuous data coverage. This is especially relevant to SPHEREx observation timelines.  
   https://ntrs.nasa.gov/citations/20240015117

5. **IBM Carbon Design System — dashboards and data visualization** — recommends limiting non-essential metrics, using consistent series colors, whitespace, linked visualizations, and purposeful annotations. HELIOS-X applies these principles to keep scientific views focused and interpretable.  
   https://www.carbondesignsystem.com/building-blocks/data-visualization/dashboards

6. **W3C WCAG 2.2** — provides the accessibility reference for text contrast, use of color, target size, and other interaction requirements. The specification uses WCAG 2.2 AA as its design target.  
   https://www.w3.org/TR/WCAG22/

---

**End of document.**
