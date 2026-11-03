# HELIOS-X Frontend Engineering Specification

**File:** `FRONTEND.md`  
**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Version:** 2.0 — Integrated Public Explorer + User Console + Admin Console
**Status:** Updated MVP implementation specification  
**Frontend stack:** React, TypeScript, Vite, Tailwind CSS  
**Backend:** FastAPI, versioned REST/JSON API at `/api/v1`  
**Primary experience:** Public SPHEREx Sky Image Explorer, with an integrated Agentic Investigation Engine  
**Frontend surfaces:** `frontend/` Public Explorer; `user-console/` authenticated research workspace; `admin-console/` restricted operations
**Design direction:** Midnight Observatory — dark-first, with a deliberately designed light theme

> This document describes what the frontend team should build and how it should integrate with the backend. It is not a claim that the UI, API endpoints, or science capabilities are already implemented. Use the relevant source-of-truth documents and the actual backend OpenAPI contract during implementation.

---

## 1. Purpose

This document defines the implementation requirements for HELIOS-X's three separately owned frontend applications. It must remain aligned with the NASA SPHEREx challenge and the current product architecture.

1. **Public SPHEREx Sky Image Explorer — `frontend/`.** The primary public-facing NASA challenge experience. Visitors enter Right Ascension (RA) and Declination (Dec), select a supported field of view (FOV), discover actual SPHEREx observations for that sky region, and inspect available image frames in actual observation-time order through a timeline and video-like viewer.
2. **Authenticated User Console — `user-console/`.** The investigation workspace for creating and managing investigations, starting and monitoring executions, inspecting observations/evidence/candidates, and reviewing scientific reports.
3. **Restricted Admin Console — `admin-console/`.** A separate operational interface for approved administrative capabilities. Its architectural boundary is defined, but it is not a NASA P0 deliverable and must not be filled with non-functional controls.

The Public Explorer must work independently of the investigation engine. Searching a sky region, loading images, changing frames, blinking images, or playing a sequence must not create an investigation, execution, agent run, candidate, or report. A user must explicitly choose to investigate a selected region/source before the investigation workflow begins.

**NASA challenge context:** Since 2025, SPHEREx has been mapping the whole sky on an approximately six-month cadence in 102 near-infrared spectral bands, producing imagery for more than a billion objects. The frontend's primary job is to let anyone enter a sky-region coordinate and inspect actual returned images over time. The mission-level cadence and large object count must not be presented as a guarantee of repeated, compatible observations for every coordinate.

The Explorer provides visual inspection, not automatic scientific proof. Playback or blinking can reveal apparent changes; scientific motion, variability, source identity, or a candidate claim requires the validated computational methods and evidence gates defined by the scientific documents.

### 1.1 MVP goals

The frontend platform must:

1. Deliver the P0 coordinate-to-observation-to-frame experience in the Public Explorer.
2. Accept valid RA/Dec inputs, supported coordinate formats, and a capability-bounded FOV.
3. Discover and display only observations and image frames actually returned by the backend/archive adapter.
4. Display a versioned frame manifest grouped into compatible sequences and ordered by actual observation/acquisition timestamps.
5. Provide frame navigation, play/pause, timeline scrubbing, timestamp display, and compatible-frame blinking without generating fake video frames.
6. Expose frame metadata, product/release context, quality and contamination information when supplied by the source/API.
7. Support an explicit Explorer-to-Investigation handoff using stable server references and a validated immutable source snapshot.
8. Preserve the investigation-to-report experience in the separate User Console: create investigation → start run → monitor progress → inspect evidence/candidates → view report.
9. Use the backend API as the source of truth for queries, manifests, artifacts, investigations, executions, tasks, evidence, candidate assessments, reports, and capability states.
10. Keep public browsing independent of authentication and agent execution, subject to backend rate/resource limits.
11. Keep authenticated/private investigation data and administrative operations behind their backend authorization boundaries.
12. Avoid fabricated observations, invented progress, unverified scientific results, duplicate submissions, stale resource views, or false success messages.
13. Remain usable on desktop, tablet, and mobile, with keyboard-operable viewer controls and reduced-motion support.
14. Follow `UI_UX_SPECIFICATION.md` for the existing User Console design language and this document's Explorer-specific interaction requirements.
15. Test the Explorer path, investigation handoff, investigation-to-report path, important failure/coverage states, and the five NASA challenge targets.

### 1.2 Explicit non-goals for the MVP

Do not implement or present these as working features unless the backend, persistence, capability registry, and tests support them end to end:

- Requiring login or an investigation record merely to browse public SPHEREx images.
- Invoking an AI agent or creating an investigation for a normal Explorer query or playback action.
- Creating synthetic frames, synthetic timestamps, fabricated observations, or regular six-month sequences to fill coverage gaps.
- Treating the 102 spectral channels as 102 observation epochs, or treating detector arrays/broad bands as time points.
- Claiming that every sky coordinate has two or more compatible observations because the mission maps the sky on an approximately six-month cadence.
- Treating visual movement, blinking, pixel displacement, or a catalogue no-match as a validated scientific motion result or discovery.
- Arbitrary WCS processing, scientific image subtraction, stacking, detector-array combination, spectral extraction, Gaia/WISE overlays, or advanced source tracking when the current deployment does not advertise the capability.
- Sending full FITS files, image arrays, base64 scientific payloads, or arbitrary artifact URLs through investigation-creation requests.
- Connecting a browser directly to PostgreSQL or exposing database credentials, storage paths, or provider credentials.
- PDF export, WebSocket/SSE updates, human-review submission, collaboration, or admin operations that are not implemented and tested.
- A “Planet X confirmed” badge, discovery animation, or any presentation that implies an automated candidate is a confirmed planet.

If a feature is planned but unavailable, omit it or mark it unavailable with a useful reason derived from the API capability response. Do not imply it is operational.

## 2. Source-of-Truth Documents

This document coordinates frontend implementation across the three applications. It does not replace or silently modify the contracts owned by the following project documents.

| Document | How the frontend uses it |
|---|---|
| `PRD.md` | Product identity, NASA challenge requirements, users, P0/P1 scope, user journeys, and non-goals |
| `SYSTEM_ARCHITECTURE.md` | Three frontend surfaces, access boundaries, component ownership, repository layout, and end-to-end data flow |
| `AGENT_SPECIFICATIONS.md` | Eight logical agent roles, their responsibilities, safe labels, and execution boundaries |
| `SCIENTIFIC_METHODOLOGY.md` | Scientific meaning, channel/epoch distinctions, uncertainty, quality, evidence, and limitations |
| `INVESTIGATION_WORKFLOW.md` | Investigation lifecycle, task dependencies, source snapshot handoff, statuses, and evidence gates |
| `IMPLEMENTATION_PLAN.md` | Repository conventions and implementation sequence |
| `DATA_SOURCE_SPEC.md` | SPHEREx products/releases, observation/frame semantics, metadata, retrieval, and artifact limitations |
| `DATABASE_SCHEMA.md` | Backend persistence model only; frontend must not depend on tables, ORM models, or database fields |
| `API_SPECIFICATION.md` | Exact endpoints, request/response schemas, status codes, query/manifest semantics, errors, and pagination |
| `TESTING_AND_ACCEPTANCE.md` | Required scientific, API, Explorer, target, and release acceptance tests |
| `SECURITY.md` | Public/private access, authentication, authorization, artifact delivery, and untrusted-data requirements |
| `OPERATIONS.md` | Local development, API origin, environment setup, storage, and deployment configuration |
| `UI_UX_SPECIFICATION.md` | Existing investigation-workspace screens, navigation, visual tokens, responsive behavior, and User Console UX |
| `BACKEND.md` | Backend services, public Explorer endpoints, artifact service, database/API integration, and workflow behavior |
| `FRONTEND.md` | Frontend application boundaries, implementation, integration, and acceptance rules (this document) |

### 2.1 Contract ownership and conflict rule

- `API_SPECIFICATION.md` and the backend's generated OpenAPI contract control API methods, paths, payloads, error codes, and response types.
- `SYSTEM_ARCHITECTURE.md` controls which product surface owns each route/feature and the public/private boundary.
- `UI_UX_SPECIFICATION.md` currently specifies the investigation workspace. Its six investigation screens belong to `user-console/`, not to the public `frontend/` application. The Public Explorer interaction requirements are specified in Section 9 of this document until a dedicated Explorer UI/UX document is approved.
- Scientific status and meaning must follow `SCIENTIFIC_METHODOLOGY.md`, `INVESTIGATION_WORKFLOW.md`, and actual API data.
- `DATABASE_SCHEMA.md` establishes PostgreSQL as the backend's final primary relational database. Frontend applications must access data only through the backend API; they must never connect to PostgreSQL directly or infer public API fields from database columns.
- `shared/` may contain stable machine-readable contracts and generated types. It must not become a second independently edited source of API truth.
- If an API contract, generated OpenAPI schema, or frontend implementation disagrees with another document, resolve the conflict with the owning document before adding a frontend-only workaround.
- Never present a planned feature as implemented simply because it appears in an architecture, API, or UI specification.

## 3. Technology and Project Setup

### 3.1 Required baseline

All three product surfaces use the same approved frontend baseline while remaining separately buildable applications:

- React.
- TypeScript with strict type checking enabled.
- Vite.
- Tailwind CSS, mapped to shared design tokens where appropriate.
- React Router or an equivalent explicit client-side router.
- A typed API client built on `fetch` or the team's selected HTTP client.
- One consistent approach to server state, caching, invalidation, and polling.
- A frontend test runner and DOM/component testing library selected by the repository.
- A browser end-to-end test tool for the Public Explorer journey and the User Console investigation journey.
- ESLint, formatting, and type-check commands defined in workspace package scripts.

The canonical repository is a monorepo with separate root folders for `frontend/`, `user-console/`, and `admin-console/`. Follow the root workspace/package-manager decision in `SYSTEM_ARCHITECTURE.md`; the architecture baseline uses a pnpm workspace. Pin dependencies and commit the lock file. Do not create a second package manager or duplicate dependency roots without an approved reason.

The three applications may share design tokens, stable generated API types, and small presentational primitives. They must not share feature state or blur their access boundaries.

### 3.2 Canonical frontend repository layout

The root repository layout is owned by `SYSTEM_ARCHITECTURE.md`. The frontend-facing portion is:

```text
helios-x/
├── package.json
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
├── frontend/                           # Public SPHEREx Sky Image Explorer
│   ├── package.json
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── src/
│       ├── app/
│       ├── routes/
│       ├── features/
│       │   ├── sky-search/
│       │   ├── sky-viewer/
│       │   ├── observation-timeline/
│       │   ├── frame-comparison/
│       │   ├── frame-metadata/
│       │   ├── capabilities/
│       │   ├── shareable-query/
│       │   └── investigation-handoff/
│       ├── components/
│       ├── services/
│       ├── hooks/
│       ├── types/
│       ├── utils/
│       ├── styles/
│       └── main.tsx
├── user-console/                       # Authenticated investigation workspace
│   ├── package.json
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── src/
│       ├── app/
│       ├── routes/
│       ├── features/
│       │   ├── workspace/
│       │   ├── investigations/
│       │   ├── execution-monitor/
│       │   ├── evidence/
│       │   ├── candidates/
│       │   ├── reports/
│       │   ├── capabilities/
│       │   ├── human-review/
│       │   └── settings/
│       ├── components/
│       ├── services/
│       ├── hooks/
│       ├── types/
│       ├── utils/
│       ├── styles/
│       └── main.tsx
├── admin-console/                      # Restricted operations/admin application
│   ├── package.json
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── src/
│       ├── app/
│       ├── routes/
│       ├── features/                    # Add only approved operational features
│       ├── components/
│       ├── services/
│       ├── hooks/
│       ├── types/
│       ├── styles/
│       └── main.tsx
└── shared/
    ├── contracts/                      # Stable cross-boundary schemas only
    ├── generated-types/                # Generated from OpenAPI where practical
    ├── design-tokens/                  # Shared tokens if package ownership is approved
    └── ui/                             # Optional small accessible primitives
```

This is the frontend-facing target layout, not a requirement to create every folder immediately. Do not put User Console investigation features under `frontend/src/features/`; `frontend/` means the Public Explorer. Do not build a full Admin Console merely because its directory is reserved in the architecture.

### 3.3 Application and module ownership

- **`frontend/`:** public coordinate search, FOV controls, Explorer query state, coverage display, observation/frame selection, image viewer, timeline/playback, metadata, supported visual comparison, shareable query URLs, and explicit investigation handoff.
- **`user-console/`:** workspace, investigation creation, execution monitor, evidence/candidate views, scientific reports, human-review UI when enabled, and user settings.
- **`admin-console/`:** only approved, role-protected operational screens backed by explicit admin API contracts. Operational features remain out of scope until approved and implemented.
- **`shared/contracts/` and generated types:** stable shared contracts and generated TypeScript types. The backend OpenAPI contract remains authoritative.
- **`shared/ui/`:** optional reusable accessible primitives. Do not share app-level navigation or feature state when it would blur product boundaries.
- **`services/` in each app:** API client and endpoint functions for that surface. Keep API requests out of presentational components.

### 3.4 Local setup

Install the workspace dependencies from the repository root using the committed lock file:

```bash
pnpm install --frozen-lockfile
```

Run an application using its actual workspace package name and scripts, for example:

```bash
pnpm --filter <explorer-package-name> dev
pnpm --filter <user-console-package-name> dev
pnpm --filter <admin-console-package-name> dev
```

The package names and scripts must be defined in the actual workspace `package.json` files; these examples are conventions, not proof that the scripts currently exist. Use the app-specific README or root development guide for ports and concurrent development. Do not assume all three applications share one route namespace or deployment origin.

Create local environment files from the corresponding `.env.example` files when environment variables are used. Only public configuration belongs in frontend environment variables. A Vite variable exposed to the browser (normally prefixed with `VITE_`) is public. Never put LLM keys, archive credentials, PostgreSQL credentials, storage credentials, access tokens, or other secrets in client-side code.

Suggested public configuration shape:

```dotenv
VITE_API_BASE_URL=http://127.0.0.1:8000/api/v1
```

Use the variable name actually selected by the implementation. Each application may use the same API origin, but public and authenticated requests must follow their respective backend access policies.

## 4. Application Architecture

The three frontend applications are separately owned and buildable. They communicate with the same FastAPI backend through the versioned public API; none imports backend Python code, connects directly to PostgreSQL, or accesses artifact storage directly.

```text
Public visitor                  Authenticated researcher           Administrator
      |                                    |                              |
      v                                    v                              v
frontend/                           user-console/                  admin-console/
Public Sky Explorer                 Investigation workspace        Restricted operations
      |                                    |                              |
      +------------------------------------+------------------------------+
                                           |
                                           | HTTPS + JSON /api/v1
                                           v
                                    FastAPI Backend
                                           |
                  +------------------------+------------------------+
                  |                        |                        |
                  v                        v                        v
          Explorer Services        Investigation Services       Admin Services
                  |                        |                        |
                  |                        v                        |
                  |                Workflow Orchestrator             |
                  |                        |                        |
                  |                Eight logical agents              |
                  |                        |                        |
                  +------------------------+------------------------+
                                           |
                         +-----------------+------------------+
                         |                                    |
                         v                                    v
               Deterministic Scientific Engine       Data-source adapters
                         |                                    |
                         +-----------------+------------------+
                                           |
                         +-----------------+------------------+
                         |                                    |
                         v                                    v
                 SQLAlchemy repositories          Controlled artifact service
                         |                                    |
                         v                                    v
                    PostgreSQL                         Private file/object storage
```

PostgreSQL is the backend's final durable relational system of record, as defined by `DATABASE_SCHEMA.md`. Large FITS/image/spectrum/report binaries remain in controlled artifact storage. The browser communicates only with FastAPI and approved artifact-content endpoints.

### 4.1 Backend is authoritative

The backend is the source of truth for:

- Explorer query IDs, query status, query outcome, normalized coordinates, and coverage summaries.
- Observation records, frame manifests, manifest versions, sequence grouping, actual observation times, and frame metadata.
- Artifact metadata, availability, content access, and derivation provenance.
- Supported FOV bounds, data releases, product types, filters, and scientific capabilities.
- Investigation and execution IDs, source snapshot IDs, task list, and canonical statuses.
- Observations, measurements, evidence, quality checks, candidate assessments, and reports.
- Whether cancellation, retry, report generation, review, or investigation handoff is permitted.

The frontend may own temporary UI state such as open panels, selected frame, selected compatible sequence, playback speed, open tab, sort selection, unsaved form values, and theme preference. It must not invent persistent scientific state, archive availability, or computed measurements.

### 4.2 Typed contract

- Prefer generated TypeScript types based on the backend OpenAPI specification, or manually maintained types with automated contract checks.
- API JSON field names remain `snake_case` unless an explicit mapping layer is used.
- Treat optional and nullable fields as distinct.
- Treat empty arrays, omitted fields, and `null` according to the API schema.
- Handle unknown future enum values defensively.
- Never assume an optional value exists because it was present in a fixture.
- Do not use raw ORM/table representations in frontend models.
- Do not create frontend-only statuses that imply a backend transition occurred.
- Do not use a preview artifact as an authoritative scientific input.

## 5. Routes and Navigation

The three applications have separate route trees. Their deployment origins or mount paths are deployment decisions; do not assume that all clients share one SPA router.

### 5.1 Public Explorer — `frontend/`

The Public Explorer opens directly into the sky-region search/viewer experience. A minimal route set is sufficient for the MVP:

| Frontend route | Purpose |
|---|---|
| `/` | Public SPHEREx Sky Image Explorer |
| `/?ra=...&dec=...&fov_width=...&fov_height=...` | Optional shareable, validated sky-region state |
| `/query/:queryId` | Optional route for restoring a public query/result when supported by the application and API |

Only supported non-sensitive query parameters may be serialized into a shareable URL. Do not include credentials, private investigation IDs, source-snapshot IDs, reviewer information, or other private data. Query parameters are input state, not proof that a query has succeeded; the frontend must validate them and ask the backend to run/restore the query.

### 5.2 User Console — `user-console/`

The existing investigation-workspace routes belong to the authenticated User Console:

| Frontend route | Purpose |
|---|---|
| `/workspace` | Active/recent investigations and actions needing attention |
| `/investigations/new` | New investigation form |
| `/investigations/:investigationId` | Investigation overview and monitor |
| `/investigations/:investigationId/evidence` | Evidence and candidates |
| `/investigations/:investigationId/reports` | Reports for an investigation |
| `/reports/:reportId` | Specific report viewer |
| `/capabilities` | Supported sources, products, and analysis operations |
| `/settings` | Theme and basic display preferences |

### 5.3 Admin Console — `admin-console/`

Admin Console routes must be defined only when the corresponding admin API, role/permission model, and operational use case are approved. Do not copy public or user routes into the Admin Console or invent routes for features that do not exist.

### 5.4 Navigation behavior

- Keep Public Explorer navigation focused on sky search, results, viewer controls, metadata, and an explicit investigate action.
- Keep User Console navigation focused on Workspace, Investigations, Capabilities, and Settings, with contextual investigation tabs for Monitor, Evidence & Candidates, and Reports.
- Admin Console navigation is separate and restricted.
- Preserve current resource context within each application.
- Browser back/forward and direct URLs must work.
- Refreshing a nested route must reload resource state from the API.
- Provide breadcrumbs where they help orientation.
- Show explicit not-found/error states for missing resources; never leave a blank page that looks successful.
- Warn about unsaved form changes before navigation when practical.
- Do not add global search, notifications, profile/account menus, or collaboration UI without real backend functionality.

### 5.5 App shell

Desktop:
- Use the visual shell appropriate to each application; do not force the investigation sidebar onto the public Explorer viewer if it reduces image workspace.
- Keep the current sky region, query state, or investigation identity visible where relevant.
- Keep primary actions close to their context.
- Allow technical tables/panels to scroll within bounded regions.

Tablet/mobile:
- Use compact navigation without hiding the sky viewer or critical timeline controls.
- Stack filters and content regions vertically.
- Keep frame navigation, pause/play, timestamp, and investigation handoff reachable.
- Avoid unreadably small labels or controls that cover the image.

## 6. Visual Design System — Midnight Observatory

Use `UI_UX_SPECIFICATION.md` as the visual source of truth. Build a restrained, polished research interface. Scientific data and legibility take priority over visual effects.

### 6.1 Dark theme tokens

| Token | Hex | Usage |
|---|---|---|
| `color.bg` | `#080D1A` | Main application background |
| `color.surface` | `#111A2D` | Panels and cards |
| `color.surface-elevated` | `#18233A` | Menus, dialogs and elevated panels |
| `color.border` | `#2B3852` | Dividers, borders and outlines |
| `color.text-primary` | `#F3F6FF` | Main text |
| `color.text-secondary` | `#AAB6D0` | Supporting text |
| `color.text-muted` | `#8391AD` | Muted labels, after contrast checks |
| `color.primary` | `#777BFF` | Primary action and selected states |
| `color.primary-hover` | `#9295FF` | Primary hover |
| `color.accent` | `#58D9E8` | Active analysis and informational emphasis |
| `color.annotation` | `#E9BE76` | Limited annotations and warnings |
| `color.success` | `#43C99A` | Successful/healthy states |
| `color.warning` | `#E9BE76` | Warning states |
| `color.error` | `#FF747C` | Error/failed states |
| `color.info` | `#58D9E8` | Informational states |
| `color.neutral-status` | `#9AA9C5` | Queued, inactive or unknown states |

### 6.2 Light theme tokens

| Token | Hex | Usage |
|---|---|---|
| `color.bg` | `#F4F6FC` | Main application background |
| `color.surface` | `#FFFFFF` | Panels and cards |
| `color.surface-elevated` | `#FFFFFF` | Menus/dialogs with border or shadow differentiation |
| `color.border` | `#DEE4F0` | Dividers and outlines |
| `color.text-primary` | `#172033` | Main text |
| `color.text-secondary` | `#4F5D78` | Supporting text |
| `color.text-muted` | `#65738D` | Muted labels after contrast checks |
| `color.primary` | `#5559E8` | Primary actions/selected navigation |
| `color.primary-hover` | `#4448D1` | Primary hover |
| `color.accent` | `#087F91` | Links and selected data highlights |
| `color.annotation` | `#8A5B12` | Warnings/annotations |
| `color.success` | `#13795B` | Success labels |
| `color.warning` | `#8A5B12` | Warning labels |
| `color.error` | `#B42332` | Error labels |
| `color.info` | `#087F91` | Informational labels |
| `color.neutral-status` | `#596984` | Queued, inactive or unknown states |

### 6.3 Color and typography rules

- Indigo is for primary actions and selection.
- Cyan is for active analysis or informational emphasis.
- Gold is reserved for warnings and scientific annotations; do not highlight everything in gold.
- Success/error colors are semantic, not decorative.
- Never use color as the only way to communicate a status. Include a text label and, when helpful, an icon.
- Do not add gradients or glows to every card. Avoid decoration that competes with evidence.
- Use Inter or a system sans-serif fallback; do not depend on an unavailable font.
- Verify text, controls and meaningful UI boundaries against WCAG 2.2 AA contrast targets.
- Charts, if supported by real data, must have labels, units, legends, uncertainty indicators where relevant, and a table/text alternative.

### 6.4 Theme implementation

Implement `system`, `dark` and `light` preference only if supported in the settings design. Use CSS variables as the canonical tokens; map Tailwind utilities to those variables instead of hardcoding hex values throughout components.

Example token file:

```css
:root {
  color-scheme: dark;
  --color-bg: #080D1A;
  --color-surface: #111A2D;
  --color-surface-elevated: #18233A;
  --color-border: #2B3852;
  --color-text-primary: #F3F6FF;
  --color-text-secondary: #AAB6D0;
  --color-primary: #777BFF;
  --color-accent: #58D9E8;
  --color-annotation: #E9BE76;
  --color-success: #43C99A;
  --color-warning: #E9BE76;
  --color-error: #FF747C;
}

[data-theme="light"] {
  color-scheme: light;
  --color-bg: #F4F6FC;
  --color-surface: #FFFFFF;
  --color-surface-elevated: #FFFFFF;
  --color-border: #DEE4F0;
  --color-text-primary: #172033;
  --color-text-secondary: #4F5D78;
  --color-primary: #5559E8;
  --color-accent: #087F91;
  --color-annotation: #8A5B12;
  --color-success: #13795B;
  --color-warning: #8A5B12;
  --color-error: #B42332;
}
```

This is a starter example. Add missing tokens (focus rings, shadows, spacing, radii and typography scale) consistently instead of mixing one-off values.

---

## 7. Shared Components

Build a small consistent component set. Each component should have a documented contract, typed inputs, and accessible behavior. The Public Explorer and User Console may share basic primitives, but feature components remain owned by their respective applications.

### 7.1 Core UI primitives

- Button and icon button.
- Text input, textarea, select, checkbox, and supported date/time fields.
- Tabs, breadcrumb, and tooltip.
- Dialog/confirmation dialog.
- Table, pagination, and sorting controls where backed by data.
- Skeleton/loading indicator.
- Alert and inline validation message.
- Empty state and error panel.
- Copyable resource ID/coordinate.
- Theme switch.
- Accessible navigation item.

### 7.2 Public Explorer components

Create the following components as the corresponding P0 viewer behavior is implemented:

- `CoordinateInput`: RA/Dec entry with explicit units, coordinate convention, supported decimal/sexagesimal formats, validation, and accessible error messages.
- `FOVSelector`: supported region shape and FOV dimensions with angular units and bounds sourced from the capability contract.
- `ExplorerQueryForm`: query filters that are actually supported by the current backend deployment.
- `CoverageSummary`: actual query status, result outcome, observation/frame counts, warnings, and coverage gaps.
- `ObservationSequenceSelector`: selects a compatible manifest sequence; it must not merge different channel/band/array/product groupings into a false time sequence.
- `FrameViewer`: displays a selected artifact-backed image and retains the frame's stable ID and metadata context.
- `FrameTimeline`: selects only returned frames from the chosen sequence, ordered by actual observation time.
- `PlaybackControls`: play/pause, previous/next, speed, and current timestamp; playback advances through available frames only.
- `FrameMetadataPanel`: actual acquisition time, product/release, wavelength/channel, detector array, WCS, units, calibration and quality metadata when available.
- `QualityFlagList`: displays source-provided quality/contamination flags without inventing meanings.
- `ArtifactImage`: retrieves image content through the controlled artifact endpoint; it does not accept arbitrary provider URLs or filesystem paths.
- `InvestigateSelectionAction`: explicit handoff action using query/manifest/frame/observation IDs.
- `ShareQueryControl`: copies or opens a URL containing only supported public query parameters.

### 7.3 Scientific and User Console components

- `StatusBadge`: consistent presentation for documented statuses.
- `QualityOutcomeLabel`: quality status plus its reason.
- `EvidenceReference`: navigable evidence identifier.
- `MeasurementValue`: value, unit, and uncertainty when available.
- `ProvenancePanel`: source, product release, originating task, method/version, and relevant timestamps.
- `LimitationNotice`: missing prerequisites, uncertainty, or scope limitation.
- `TaskRow`: task role/name, status, attempt, timestamp, and concise result/error.
- `CandidateStatusPanel`: canonical candidate status, supporting/contradictory evidence, alternative explanations, and missing data.
- `ReportSectionNav`: links to actual report sections.
- `TechnicalDetailsDisclosure`: collapsible structured metadata, JSON, or diagnostics.

Use the same status label, semantic color, and icon consistently across applications. Keep technical details available without forcing every user to read raw JSON.

## 8. Status and Scientific Result Presentation

Use canonical values returned by the backend; map them to human-readable labels in one shared status-mapping module. Do not combine operation status, result outcome, quality outcome, evidence-gate outcome, and candidate status into a single badge or enum.

### 8.1 Explorer query and retrieval states

Explorer query/job resources reuse the canonical operation/task-style statuses:

- `QUEUED`
- `RUNNING`
- `SUCCEEDED`
- `FAILED`
- `RETRYING`
- `CANCELLED`
- `BLOCKED`
- `NOT_SUPPORTED`
- `INSUFFICIENT_DATA`

A separate `outcome_code` describes the completed query result. The current API contract may return values such as:

- `MULTIPLE_OBSERVATIONS`
- `SINGLE_OBSERVATION`
- `NO_MATCHING_OBSERVATIONS`
- `PARTIAL_RESULTS`
- `PRODUCT_UNAVAILABLE`
- `INSUFFICIENT_METADATA`
- `UNSUPPORTED_QUERY`
- `INVALID_PRODUCT`
- `PROVIDER_ERROR`

These outcome codes are not task statuses. Use the exact API enum and handle unknown future values defensively.

Important UI semantics:

- A successful query with zero records is `SUCCEEDED` with `NO_MATCHING_OBSERVATIONS` and a zero count. It is not a provider failure and does not mean no astronomical object exists in the sky region.
- One available observation is a valid result; explain that a temporal sequence cannot be formed from one frame.
- Partial results must show which records/frames are available and the API-provided warnings about missing coverage.
- A failed poll request is a temporary refresh problem, not an Explorer query failure.
- Query status and retrieval outcome must remain distinct from scientific detection or analysis status.

### 8.2 Task statuses

The project baseline is:

- `QUEUED`
- `RUNNING`
- `SUCCEEDED`
- `FAILED`
- `RETRYING`
- `CANCELLED`
- `BLOCKED`
- `NOT_SUPPORTED`
- `INSUFFICIENT_DATA`

Execution-level states may include `CANCEL_REQUESTED` and other values defined by the API. Do not assume the task enum and execution enum are identical.

Suggested human-readable labels:

| API status | UI label guidance |
|---|---|
| `QUEUED` | Queued |
| `RUNNING` | Running |
| `SUCCEEDED` | Completed |
| `FAILED` | Failed |
| `RETRYING` | Retrying |
| `CANCELLED` | Cancelled |
| `BLOCKED` | Blocked |
| `NOT_SUPPORTED` | Not supported |
| `INSUFFICIENT_DATA` | Insufficient data |
| Unknown future value | Unrecognized status — show raw value in details |

A temporary polling/API error is **not** evidence that a task failed. Show it as a data-refresh error while preserving the last known backend state and retrying safely where appropriate.

### 8.3 Quality outcomes

- `PASSED`
- `FAILED`
- `INCONCLUSIVE`
- `NOT_AVAILABLE`
- `NOT_APPLICABLE`

Do not collapse these into a simple green/red indicator. Display the text and reason where available.

### 8.4 Evidence gates

- `PASSED`
- `FAILED`
- `UNKNOWN`
- `NOT_APPLICABLE`

`UNKNOWN` is not passed. A not-applicable gate must include the backend-provided reason where available.

### 8.5 Candidate statuses

- `NEEDS_MORE_DATA`
- `NOT_PRIORITIZED`
- `PRIORITIZED_FOR_REVIEW`
- `BLOCKED_BY_QUALITY_ISSUE`

`PRIORITIZED_FOR_REVIEW` means a candidate merits human review. It does **not** mean a discovery or confirmed planet. There must be no celebratory discovery animation or confirmed-planet badge.

### 8.6 Scientific values and time/channel semantics

- Display units with values.
- Show uncertainty only when supplied; if unavailable, say so rather than showing zero.
- Preserve quality flags and contradictory evidence.
- Distinguish observed data, computed measurements, modelled/derived values, interpretation, and hypotheses.
- Clearly label synthetic/sample data.
- Show only charts supported by valid data and a supported method. If a plot is unavailable, show a table or metadata instead.
- Sort frame timelines by the actual observation/acquisition timestamp supplied by the frame manifest, not retrieval time, file time, release time, channel number, or detector-array number.
- Keep observation epoch, spectral channel/wavelength, broad wavelength band, detector-array identity, release, processing version, and retrieval time as distinct fields/dimensions.
- The 102 spectral channels are not 102 time epochs. The six detector arrays are not six observation dates. Never map either dimension to the temporal playback axis.
- A mission-level six-month sky-survey cadence is not a promise of two compatible observations for every target or sky region.

## 9. Public Explorer — Screen-by-Screen Requirements

The Public Explorer lives in the root `frontend/` application and is the primary public-facing NASA challenge deliverable. It must remain independently usable without login, an investigation record, or an AI-agent run. Public query/artifact operations remain read-only with respect to scientific source data and are subject to backend rate, FOV, result-count, concurrency, and download-size limits.

### 9.1 Explorer home and page structure

The default route opens the coordinate search and sky-image workspace. The page should prioritize the image itself and its timeline rather than an investigation dashboard.

Recommended layout:

- **Search/header region:** RA, Dec, coordinate convention, FOV, and a clear “Search sky region” action.
- **Optional filter region:** observation-time range, product release, spectral channel/wavelength, broad band, detector array, or product type only when returned as supported by the capability contract.
- **Coverage region:** query state, returned observation/frame counts, selected sequence, warnings, and missing-coverage explanation.
- **Viewer region:** selected frame image, timestamp, and previous/next/play/pause controls.
- **Timeline region:** frames from the currently selected compatible sequence only.
- **Metadata region:** selected-frame source/product/release and available quality/WCS information.
- **Explicit investigation action:** “Investigate this region/source” or equivalent, enabled only when a valid selection and backend handoff capability exist.

Do not place Workspace, agent monitoring, candidate ranking, or report cards in the Public Explorer shell. Those belong to the User Console.

### 9.2 Coordinate input

The challenge requires users to enter RA and Dec for a sky patch.

- Provide clearly labelled RA and Dec inputs with units and accepted coordinate format.
- Support decimal degrees and sexagesimal entry only when the parser/validator is implemented and tested. Do not silently reinterpret ambiguous values.
- Show the supported coordinate frame/convention, defaulting only according to the API/capability contract (for example, ICRS when explicitly supported).
- Validate finite values and documented ranges before submission; preserve user-entered values after recoverable errors.
- Keep original user input distinct from server-normalized coordinates when the API returns both.
- Do not invent target coordinates or silently substitute another object's coordinates.
- For named targets, use the optional target-resolution API only when enabled; display the resolver/source, resolved identifier, frame/epoch, and ambiguity where available.

The NASA brief supplies the following coordinates for WISE 0855: RA `08h 55m 10.831s`, Dec `−07° 14′ 42.53″`. Other named-target coordinates must be obtained from an approved resolver/catalogue or the actual challenge data; do not hardcode guessed coordinates.

### 9.3 Field-of-view selection and optional filters

- Provide a clear supported FOV control with explicit angular units and shape/dimensions allowed by the API.
- Load minimum/maximum FOV, supported shapes, and query limits from `GET /api/v1/capabilities` or the exact capability contract.
- Do not silently clamp an unsupported FOV. Show the backend's validation message or supported bounds.
- Larger FOVs may increase retrieval time and transfer size. Display estimates or warnings only when supplied by the backend/capability contract; do not invent latency or byte counts.
- Offer time-range, release, channel, band, array, and product filters only when the deployment advertises them as supported.
- Keep observation time filters separate from retrieval time, release date, and file creation time.
- Do not make empty arrays or omitted filters mean anything different from the API schema.

### 9.4 Explorer query submission and coverage

The Public Explorer uses `POST /api/v1/explorer/queries` to submit a bounded sky-region query.

- Prevent duplicate query submission while the request is pending; use the API's idempotency contract where supported.
- Treat the `202 Accepted` response as a queued query, not as completed data retrieval.
- Poll `GET /api/v1/explorer/queries/{query_id}` according to the documented API behavior; stop polling at a terminal state and back off on transient errors.
- Do not create an investigation, execution, agent run, candidate, or report for a normal query.
- Show actual coverage counts and query outcome values returned by the backend.
- Distinguish queued, running, succeeded, failed, unsupported, insufficient-data, blocked, cancelled, and partial-result states.
- A successful empty result must say that no matching observations were returned for the query criteria. It must not say that no object exists.
- A single returned observation is a valid outcome. Explain that a time sequence is unavailable or limited instead of manufacturing extra frames.
- A provider error, invalid product, unsupported filter, and insufficient metadata must remain distinct.

### 9.5 Observation list and frame-manifest sequence selection

After a query completes, retrieve observations and the versioned frame manifest using the API contract:

- `GET /api/v1/explorer/queries/{query_id}/observations`
- `GET /api/v1/explorer/queries/{query_id}/frames`

The UI must:

- Show only observations/frames present in the returned manifest.
- Display the manifest version and preserve the query/manifest IDs for later navigation and handoff.
- Allow the user to choose among sequence groups when the manifest contains multiple compatible groups.
- Keep channel/wavelength, broad band, detector array, release, processing version, and other compatibility dimensions visible in sequence details.
- Order frames inside a compatible sequence by actual observation/acquisition time ascending.
- Never sort frames by retrieval time, file timestamp, release timestamp, channel number, or detector-array number.
- Never merge incompatible channel/band/array/product groupings into one apparent time series.
- Make one-frame and zero-frame states explicit.
- Preserve stable frame IDs when the sequence selection or viewer state changes.

One observation/product may contain multiple image planes, channels, or detector-array products. The frontend must not assume one observation equals one frame or that multiple frames automatically represent multiple epochs.

### 9.6 Image viewer and time-based playback

The viewer is a presentation component over actual frame artifacts. It is not itself a scientific motion detector.

P0 controls:

- Display the currently selected frame.
- Play and pause the available sequence.
- Previous and next frame.
- Timeline scrubbing/frame selection.
- Playback speed selection, if implemented without changing the scientific timestamps.
- Show the actual timestamp and sequence context for the displayed frame.
- Provide an accessible frame counter such as “Frame 2 of 5” when counts are known.
- Allow users to stop playback and inspect one frame without losing the current selection.

Behavior rules:

- Advance only through frames actually returned by the selected manifest sequence.
- Do not generate interpolated frames, repeat missing epochs, or imply a regular cadence.
- If only one frame exists, disable sequence playback and explain why.
- If no frames exist, show a query/coverage empty state instead of a blank viewer.
- Keep actual acquisition time visible; playback speed is not observation cadence.
- Do not use animated scanning, pulsing, or motion effects that could be mistaken for source motion.
- Respect `prefers-reduced-motion`; users must be able to use the timeline without autoplay.
- Do not render a server-generated “video” unless a separate supported product explicitly provides one. The P0 viewer consumes individual frame artifacts and handles playback in the browser.

Blinking/rapid switching between two selected compatible frames may be provided as a visual aid. The UI must label it as visual comparison, not a validated motion measurement.

### 9.7 Frame metadata and artifact display

For the selected frame, display metadata when supplied by the API/source product:

- Actual observation/acquisition timestamp and time scale.
- Retrieval timestamp, labelled separately from observation time.
- Observation ID, product ID/type, source, release, and processing version.
- Spectral channel and wavelength with units.
- Broad-band and detector-array identity when present.
- Image dimensions, pixel scale, and measurement units when available.
- Coordinate frame, WCS availability/reference, and region context.
- Calibration status, quality flags, contamination flags, mask/variance artifacts, and PSF reference where available.
- Source product, cutout, and preview artifact references.

Use `GET /api/v1/explorer/frames/{frame_id}` and the controlled artifact endpoints defined in `API_SPECIFICATION.md`. Do not embed large image arrays in JSON or fetch arbitrary provider URLs directly from the browser.

Clearly distinguish:

- **Original/source product:** authoritative archive product or controlled source artifact used for scientific processing.
- **Cutout:** derived spatial subset whose coordinate/WCS relationship must be preserved where supported.
- **Preview:** browser-display derivative that may be lossy or rescaled and is not a replacement for the scientific source product.
- **Mask/variance/other auxiliary artifact:** separate data products with their own semantics.

If WCS, uncertainty, calibration, or quality metadata is unavailable, say unavailable; do not invent defaults. An image failing to load is an artifact-delivery error, not evidence that the observation did not exist.

### 9.8 Visual comparison and optional scientific operations

P0 visual comparison:

- Allow side-by-side inspection or blinking of selected frames when the manifest groups them as compatible.
- Show each frame's timestamp and identity while comparing.
- Warn when frames differ in channel, band, detector array, product, release, or another dimension relevant to a valid comparison.
- Do not describe raw pixel differences as physical motion or variability.

Conditional/P1 operations:

- Server-side difference images, addition/stacking, detector-array combination, and on-demand spectra may be exposed only when the capability API advertises them and the backend operation has passed the scientific validation requirements.
- The frontend requests operations through their documented APIs; it must not perform authoritative calibration, registration, subtraction, coaddition, source detection, or scientific measurement in ad hoc browser code.
- Keep operation inputs, method/version, warnings, output artifact, and quality diagnostics visible.
- Do not coadd across observation epochs by default; preserve original epoch frames.
- Keep six detector arrays separate unless a validated method permits combination. Do not represent detector-array combination as temporal stacking.
- Gaia and WISE/NEOWISE overlays, when enabled, must remain separately attributed context layers and must not be merged into SPHEREx image pixels or measurements.
- Optional overlays and comparison operations must not delay the required coordinate-to-frame Explorer path.
- Aladin Lite may be considered as an optional contextual sky-map/catalogue/footprint layer if its integration is approved and useful. It must not replace the custom SPHEREx frame timeline/viewer or blur the provenance of external imagery.

### 9.9 Explicit Explorer-to-Investigation handoff

The action to investigate must be explicit and separate from query, frame selection, playback, and visual comparison.

When a user chooses to investigate:

1. Confirm the selected sky region/query and selected observation/frame set.
2. Send stable references only, including `explorer_query_id`, `frame_manifest_id`, `manifest_version`, selected frame IDs, selected observation IDs, and supported analysis choices as specified by the API.
3. Create the investigation through `POST /api/v1/investigations` using the `source_context` contract.
4. Let the backend validate the query/manifest/frame references and create an immutable server-side source snapshot.
5. Display the returned `investigation_id` and `source_snapshot_id` only after the server confirms creation.
6. Navigate to the User Console or its authenticated continuation flow.
7. Start the agentic workflow only after a separate explicit user action using `POST /api/v1/investigations/{investigation_id}/runs` and the returned `source_snapshot_id`.

Access behavior:

- Public image browsing remains available without login.
- Creating or viewing a private investigation requires the authentication/access policy defined by the backend and `SECURITY.md`. If the visitor is not authenticated, offer the approved sign-in/continue path; do not invent a guest-investigation mode.
- Creating the investigation/source snapshot does not automatically start an execution or agent run.
- A client-provided frame ID is a reference, not proof that the data are still valid. The server revalidates source data and manifest version.
- Never send image arrays, base64 FITS/image content, client-calculated scientific measurements, arbitrary artifact URLs, filesystem paths, or client-edited archive metadata in the handoff payload.
- If the snapshot is stale or invalid, show the backend error and ask the user to refresh/reselect; do not silently substitute different frames or releases.
- Returning from the User Console to the Explorer should preserve the shareable public sky-region state where practical, but must not expose private investigation data in the public URL.

### 9.10 Shareable URLs and named targets

- When implemented, encode only supported public query state such as RA, Dec, coordinate frame, FOV, and optional public filters in the URL.
- On direct load, validate URL values and rerun/restore the query through the backend; do not trust cached URL values as completed query results.
- Do not put access tokens, credentials, private IDs, source snapshot IDs, user data, or confidential investigation state in URLs.
- Named-target resolution is conditional/P1. Use the documented resolver endpoint only when enabled, preserve resolver/catalogue provenance, and show ambiguity rather than silently selecting one object/component.
- If a named target is ambiguous (including component-level systems), require a clear selection or ask for coordinates; never choose a component silently.

### 9.11 NASA challenge target validation in the UI

The Public Explorer must be exercised with the five targets named in the NASA challenge:

- Barnard's Star.
- Epsilon Indi B (the brown-dwarf pair; component ambiguity must be handled where relevant).
- WISE 0855 — challenge-supplied coordinates: RA `08h 55m 10.831s`, Dec `−07° 14′ 42.53″`.
- RW Aurigae.
- R Draconis.

For targets without challenge-supplied coordinates, resolve through an authoritative supported name resolver/catalogue and preserve its identifier, coordinate frame/epoch, and source provenance. Do not hardcode guessed coordinates.

Acceptance does not require a positive detection, a moving-source result, or a variability claim. It requires that the frontend can submit the target/region, accurately show actual archive coverage and frames returned, handle ambiguity and unavailable products, and state limitations truthfully. A valid zero-result or one-observation result may pass when represented accurately.

## 10. User Console — Screen-by-Screen Requirements

The six investigation-workspace screens below are preserved from the previous frontend specification. They belong to the authenticated `user-console/` application, not the public `frontend/` Sky Image Explorer. Their visual behavior is governed by `UI_UX_SPECIFICATION.md`.

## 10.1 Workspace — `/workspace`

**Goal:** Help the user continue useful work, not look at vanity metrics.

### Layout
- Title: “Workspace” or “Your investigations”.
- Primary action: “New investigation”.
- “Continue working”: active/recent investigations ordered by meaningful activity/status.
- “Needs attention”: failed runs, blocked tasks, missing data, or actions that require attention.
- “Recent investigations”: compact searchable/filterable list if there are enough records.
- First-use state for users with no investigations.

### Investigation row/card

Show only available API data:
- Investigation name or scientific question summary.
- Target/coordinates when relevant and supplied.
- Current status.
- Last-updated time.
- Latest execution status when available.
- Useful next action such as “Open investigation” or “Inspect failure”.

### Behavior and states
- Selecting a row opens the investigation.
- “New investigation” opens the create flow.
- “Resume/Open” returns to the current resource; it must not silently start another run.
- Filters/sorting must operate on actual data.
- Provide loading skeletons, initial empty state, filtered-empty state, API error/retry and populated states.
- Do not invent totals like “Total Agents”, “Sources” or “Evidence” merely to fill dashboard space.

## 10.2 New Investigation — `/investigations/new`

**Goal:** Collect a scientifically meaningful request while allowing only capabilities supported by the active backend.

Use a short form or compact stepper, not an unnecessarily long wizard.

### Form sections

**A. Scientific question**
- Investigation name.
- Scientific question/objective.
- Optional notes where supported.

**B. Target and search parameters**
- Supported target type, such as sky position, only if the backend capability supports it.
- Required target parameters.
- Clear coordinate system, units and range validation.
- Preserve the user-entered values after recoverable API errors.

**C. Data source and analysis**
- Load supported source/product/analysis options from the capabilities endpoint.
- Clearly show requirements and limitations.
- Disable or omit unsupported options.
- Do not enable temporal/motion analysis unless the backend reports support and its prerequisites can be met.
- Do not label 102 spectral channels as temporal epochs.

**D. Review and submit**
- Summarize the investigation request before submission.
- Make clear that data availability and scientific validity are evaluated by the backend workflow.
- Disable duplicate submission while pending.
- Show field-level validation and server validation errors.
- Do not show a success notification until the server confirms creation.

After successful creation, navigate to the created investigation using the ID returned by the API.

### Backend alignment

The example `POST /api/v1/investigations` body in `API_SPECIFICATION.md` is illustrative; the actual Pydantic/OpenAPI schema and `/capabilities` response control the implemented fields and allowed options.

## 10.3 Investigation Monitor — `/investigations/:investigationId`

**Goal:** Show what is actually happening, what completed, and what needs attention.

### Page structure
- Investigation name/ID and scientific question.
- Aggregate execution status.
- Start/run action only when permitted.
- Real progress information.
- Ordered task list; use a dependency graph only if meaningful and readable.
- Task details panel.
- Clear access to Evidence & Candidates and Reports.

### Progress
- Show percentage only if the API supplies enough information to compute a meaningful percentage.
- Otherwise show task counts, current task, status or “Progress not available”.
- Do not animate arbitrary progress from 0% to 100%.
- Distinguish UI loading from scientific workflow progress.
- Show completed/total task counts only when available.

### Task rows
Each task should display available information:
- Task name/type and logical role.
- Canonical status.
- Attempt number/retry information when supported.
- Start/finish time when available.
- Short result/error summary.
- Selection opens details with input/output summary, method version, timestamps and retry history when supplied.

Only show agents/tasks that the backend actually created for that execution. The Workflow Orchestrator is a system component, not a ninth agent.

### Polling and actions
- Poll the documented status endpoint at a reasonable interval while execution is active.
- Pause polling when the page is hidden or execution is terminal where practical.
- Stop polling when terminal.
- Treat a failed poll as a temporary refresh problem, not a task failure.
- On refresh, re-fetch execution and tasks by ID.
- Disable or guard duplicate start actions while the request is pending.
- Cancellation should require confirmation when it may interrupt work.
- Show “Cancellation requested” until the backend confirms cancellation; do not imply cancellation completed early.
- Show retry only when the API indicates retry is permitted.
- Do not imply all eight agent roles are running when only a subset of tasks exists.

## 10.4 Evidence & Candidates — `/investigations/:investigationId/evidence`

**Goal:** Explain what was measured, how it was measured, its quality, and why a candidate is or is not prioritized.

Use sub-tabs or separated sections for:
- Observations/data products.
- Measurements.
- Evidence.
- Candidates and assessments.

Show only sections supported by the current investigation. An empty section should explain why no records are available; it should not automatically look like an error.

### Evidence list and details

Show, when available:
- Evidence title/type and summary.
- Originating task.
- Source product/observation reference.
- Method name and version.
- Quality outcome.
- Timestamp.
- Known limitations.

Selecting an evidence item opens the detail view with structured metadata, supporting measurements and provenance references. Put large technical JSON inside a disclosure panel.

### Measurement display

Each measurement should show as available:
- Quantity name.
- Value and unit.
- Uncertainty.
- Quality flags.
- Method/version.
- Input evidence references.

If uncertainty is missing, label it unavailable; never substitute zero.

### Candidate details

Show:
- Provisional designation/label.
- Canonical current status.
- Assessment version/timestamp.
- Supporting evidence.
- Contradictory evidence.
- Alternative explanations.
- Missing data and limitations.
- Suggested next steps, only if returned by the backend.

Never turn a candidate into a confirmed discovery based on an automated score or catalogue no-match.

### Visualizations

Image, spectral, motion or coordinate charts are allowed only when the actual observation type and supported method supply valid data. Include units, labels, legends and uncertainty where applicable. Provide a table/text alternative. Sample results must be visibly labelled.

## 10.5 Report Viewer — `/reports/:reportId`

**Goal:** Present a readable report while preserving evidence, uncertainty and limitations.

### Header
- Report title.
- Investigation reference.
- Report status/version.
- Generated timestamp.
- Only implemented actions, such as copying a reference or downloading a supported format.

### Content
Render the report sections returned by the API, which may include:
- Executive summary.
- Scope and data sources.
- Methods.
- Findings and measurements.
- Quality outcomes and uncertainties.
- Candidate assessment.
- Alternative explanations and unresolved issues.
- Limitations/next steps.
- Evidence/source references.

Use a contents rail for long reports on desktop and a compact section menu on smaller screens. Clicking an evidence reference should open the related evidence or the relevant evidence section. If the referenced resource is unavailable, show an error.

### Report states
- No report: explain that no report exists and offer generation only when permitted.
- Queued/running: show generation status and allow refresh/back navigation.
- Succeeded: show report content.
- Failed: show a useful error and retry only if permitted by the API.
- Incomplete/partial: label it and explain what is missing.

A report must display its version. It must not silently change content under the same version ID. PDF export must not be shown until the deployment implements and tests it. Markdown/HTML rendering may be used if supported.

## 10.6 Settings & Capabilities

### Capabilities
Load capability information from the backend capabilities endpoint. For each source, product and operation, show:
- Display name.
- Actual support state.
- Required input/data where relevant.
- Limitations/constraints.
- Experimental, unavailable or supported state.

Do not make broad capability claims based on a planned adapter. The frontend does not decide the scientific support matrix.

### Settings
Keep MVP settings minimal:
- Theme preference: system, dark or light, if implemented.
- Basic display/accessibility preferences only when implemented.

Do not add account/team management, notifications, or API-key controls without real backend functionality.

---

## 11. API Integration

`API_SPECIFICATION.md` defines the exact API. Base URL is configuration-driven and normally ends in `/api/v1`. Endpoint functions should append only the resource path, not duplicate the base prefix. The implemented backend OpenAPI contract controls production types and behavior.

### 11.1 Required endpoint groups by application

**Public Explorer — `frontend/`**

| UI use | API resource/method examples |
|---|---|
| Capability discovery | `GET /capabilities` |
| Submit sky-region query | `POST /explorer/queries` |
| Query status/coverage | `GET /explorer/queries/{query_id}` |
| Observations | `GET /explorer/queries/{query_id}/observations` |
| Versioned frame manifest | `GET /explorer/queries/{query_id}/frames` |
| Frame metadata | `GET /explorer/frames/{frame_id}` |
| Artifact metadata/content | `GET /artifacts/{artifact_id}`, `GET /artifacts/{artifact_id}/content` |
| Conditional cutout request | `POST /explorer/frames/{frame_id}/cutouts` |
| Optional comparison | `POST /explorer/comparisons`, `GET /explorer/comparisons/{comparison_id}` |
| Optional overlays | `GET /explorer/queries/{query_id}/overlays` |
| Optional named-target resolver | `POST /target-resolutions`, `GET /target-resolutions/{resolution_id}` |

**Explorer-to-Investigation handoff and User Console — `user-console/`**

| UI use | API resource/method examples |
|---|---|
| Create direct or Explorer-origin investigation | `POST /investigations` |
| Workspace list/detail/edit | `GET /investigations`, `GET/PATCH /investigations/{investigation_id}` |
| Start workflow after explicit user action | `POST /investigations/{investigation_id}/runs` |
| List runs / monitor execution | `GET /investigations/{investigation_id}/runs`, `GET /executions/{execution_id}` |
| Cancel run / list tasks | `POST /executions/{execution_id}/cancel`, `GET /executions/{execution_id}/tasks` |
| Task detail/retry | `GET /tasks/{task_id}`, `POST /tasks/{task_id}/retry` |
| Retrieval/observation | `/data-retrievals`, `/observations/{observation_id}` |
| Evidence and measurements | `/investigations/{investigation_id}/evidence`, `/evidence/{evidence_id}`, evidence measurement routes |
| Candidates and assessments | `/investigations/{investigation_id}/candidates`, `/candidates/{candidate_id}`, candidate assessment routes |
| Reports | report list/create/detail/content/version endpoints |
| Human reviews | only if review functionality is enabled and the user has reviewer permission |

**Admin Console — `admin-console/`**

Use only endpoints explicitly defined in the API specification and approved for the administrator role. Do not infer admin endpoints from backend modules or create UI controls without API operations and authorization tests.

The endpoint inventory is a contract reference, not proof that every route is live. Confirm implementation against generated OpenAPI and the capabilities endpoint. A planned or unavailable operation must not appear enabled.

### 11.2 API client responsibilities

Create a reusable API client for each app or an approved shared package that:

- Adds the configured base URL.
- Sends/accepts JSON with correct headers for JSON endpoints.
- Handles successful and error responses consistently.
- Preserves HTTP status, stable backend error code, and request ID when supplied.
- Converts network failures into a typed client error distinct from a scientific query/task failure.
- Never exposes raw stack traces or secrets in user-facing UI.
- Allows requests to be aborted where appropriate.
- Supports documented query parameters and pagination.
- Does not silently retry mutations.
- Does not contain screen-specific display logic.
- Has a documented path for controlled artifact/binary responses distinct from JSON parsing.

Example JSON client skeleton — adapt it to the actual response/error schema:

```ts
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;

export class ApiClientError extends Error {
  constructor(
    message: string,
    public readonly status?: number,
    public readonly code?: string,
    public readonly requestId?: string,
  ) {
    super(message);
    this.name = "ApiClientError";
  }
}

export async function apiRequest<T>(
  path: string,
  init: RequestInit = {},
): Promise<T> {
  if (!API_BASE_URL) {
    throw new Error("VITE_API_BASE_URL is not configured.");
  }

  const headers = new Headers(init.headers);
  if (init.body && !headers.has("Content-Type")) {
    headers.set("Content-Type", "application/json");
  }
  headers.set("Accept", "application/json");

  let response: Response;
  try {
    response = await fetch(`${API_BASE_URL}${path}`, {
      ...init,
      headers,
    });
  } catch {
    throw new ApiClientError(
      "The API could not be reached. Check your connection and try again.",
    );
  }

  const requestId = response.headers.get("X-Request-ID") ?? undefined;
  const raw = await response.text();
  let body: unknown;
  try {
    body = raw ? JSON.parse(raw) : undefined;
  } catch {
    body = raw;
  }

  if (!response.ok) {
    const payload = body as {
      error?: { code?: string; message?: string };
      request_id?: string;
      detail?: string;
    } | undefined;
    throw new ApiClientError(
      payload?.error?.message ?? payload?.detail ??
        `Request failed with HTTP ${response.status}.`,
      response.status,
      payload?.error?.code,
      payload?.request_id ?? requestId,
    );
  }
  return body as T;
}
```

This is an illustrative starting point, not a substitute for the exact API response contract. If the server returns a consistent `data` envelope, type and unwrap that envelope in the client rather than scattering envelope handling across components.

Artifact content is not necessarily JSON. Fetch or render it only through the controlled artifact-content endpoint and the documented access method. Validate content type/size where applicable, distinguish a pending/expired/failed artifact from an image decode error, and never render an arbitrary provider URL or server filesystem path.

### 11.3 Endpoint service pattern

Use small feature-oriented functions. Keep Explorer query functions separate from investigation functions so ordinary image browsing cannot accidentally call investigation creation or run endpoints.

```ts
export function submitExplorerQuery(payload: ExplorerQueryRequest) {
  return apiRequest<ExplorerQueryAcceptedResponse>("/explorer/queries", {
    method: "POST",
    body: JSON.stringify(payload),
  });
}

export function getExplorerQuery(queryId: string) {
  return apiRequest<ExplorerQueryResponse>(
    `/explorer/queries/${encodeURIComponent(queryId)}`,
  );
}

export function getFrameManifest(queryId: string) {
  return apiRequest<FrameManifestResponse>(
    `/explorer/queries/${encodeURIComponent(queryId)}/frames`,
  );
}

export function createInvestigation(payload: CreateInvestigationRequest) {
  return apiRequest<InvestigationCreateResponse>("/investigations", {
    method: "POST",
    body: JSON.stringify(payload),
  });
}
```

Types and response names above are illustrative. Import exact generated types from the active API contract. Encode path IDs safely and use documented request/response shapes. Do not put investigation-creation calls inside normal Explorer query or playback hooks.

### 11.4 Long-running Explorer queries and workflow execution

Explorer query flow:

1. Submit `POST /explorer/queries`.
2. Store the returned `query_id` and initial status.
3. Poll `GET /explorer/queries/{query_id}` under the documented policy.
4. On completion, request observations and the versioned frame manifest.
5. Fetch selected frame metadata/artifacts as needed.
6. Stop polling at a terminal state; preserve the last known state during transient network errors.

Investigation workflow flow:

1. Explicitly create the investigation/source snapshot from the selected Explorer references or create a direct investigation.
2. Wait for successful backend confirmation.
3. Only after a separate explicit user action, submit the run request; for Explorer-origin runs, include the returned `source_snapshot_id`.
4. Poll execution/task resources and stop at terminal states.

For both flows:

- Poll at a sensible interval; use backoff on transient errors.
- Pause when the browser tab is hidden and stop when the resource is terminal.
- Do not show a made-up percentage.
- Keep last known status visible during temporary refresh failure with a separate “Unable to refresh” message.
- Do not confuse an Explorer query with an investigation execution.
- Do not start an agent run merely because an investigation/source snapshot was created.

### 11.5 Duplicate requests and form behavior

- Disable or guard submit buttons while a mutation is pending.
- Use backend-supported idempotency where defined; button disabling alone is not enough.
- Do not retry a mutation automatically unless it is safe and the contract explicitly allows it.
- Ignore stale read results if the user navigated to another query, manifest, or investigation.
- Abort obsolete GET requests where practical.
- Preserve form values after recoverable failures.
- Show success only after receiving a successful backend response.
- Use a confirmation step for potentially disruptive cancellation.
- Do not offer retry when the backend has not permitted it.

### 11.6 API request and error states

The UI must distinguish:

- Client-side field validation error.
- API validation error.
- Network/server request failure.
- Explorer query failure.
- Successful query with zero observations.
- Query with one observation.
- Partial results with warnings.
- Analysis not run.
- Unsupported analysis (`NOT_SUPPORTED`).
- Insufficient data (`INSUFFICIENT_DATA`).
- Artifact pending, unavailable, expired, or failed.
- Cancel requested vs cancellation confirmed.
- Partial investigation results vs total success.

Use the backend request/correlation ID for diagnostics when available, but do not treat it as an authorization token.

## 12. Data Fetching and State Management

Choose one consistent approach for server state. A query library can simplify caching, invalidation and polling; alternatively, small feature hooks are acceptable for this MVP. Do not combine multiple competing state-management systems without need.

### Guidelines
- Treat API data as server state.
- Keep only ephemeral presentation state in local component state.
- Reload resource state from the backend after refresh.
- Invalidate/refetch related resources after confirmed mutations.
- Avoid maintaining a separate client-side “truth” for run statuses, candidates or reports.
- Keep pagination/sorting/filter state synchronized with the API contract.
- Handle missing and optional values defensively.
- Use abort controllers or query cancellation for obsolete reads.
- Avoid global state for data that belongs to one feature.

### Suggested query/mutation boundaries

- Workspace: list investigations and supported attention states.
- Investigation: retrieve detail and executions.
- Monitor: retrieve execution/task data and poll while active.
- Evidence: retrieve evidence, measurements and candidates for the selected investigation.
- Report viewer: retrieve report metadata, content and versions.
- Capabilities: retrieve and cache current deployment capabilities.
- Settings: persist theme/display preference only as supported by the design.

Do not use hardcoded mock values in production paths. If development fixtures are used, label the UI clearly and isolate them from live data mode.

### 12.3 Public Explorer query and frame state

- Keep the current Explorer query ID, query status, query outcome, manifest ID/version, selected compatible sequence, and selected frame as separate state values.
- Key cached results by normalized query inputs and query ID; key frame data by manifest ID/version and frame ID.
- Invalidate/refetch observations and manifests only when the API indicates the query/manifest has changed or a new query is submitted.
- Changing the selected frame or playback position is ephemeral UI state; it must not start a new archive query unless a new query parameter is submitted.
- Do not build a local frame manifest from guessed dates, file names, or channel numbers.
- Do not merge incompatible manifest sequences in client state.
- Cache artifact content only under a bounded policy that respects artifact expiry and authorization. Do not persist large FITS/image binaries in local storage.
- Keep Public Explorer query state separate from User Console investigation state. An explicit handoff creates the link between them through the API's source snapshot.

### 12.4 Cross-application state boundary

- A route or local-storage value from the Public Explorer is not an authenticated investigation record.
- The User Console reloads private resources from the backend after navigation or refresh.
- Shared generated types do not imply shared client-side ownership of server state.
- The Admin Console must not reuse User Console permissions or trust UI-only role flags.

---

## 13. Forms, Validation and Scientific Input UX

- Client-side validation helps the user but does not replace backend validation.
- Reuse backend constraints where practical and keep the API authoritative.
- Show inline field errors near the field.
- Explain expected coordinate conventions and units.
- Reject malformed numeric values before submission where the form schema permits.
- Do not silently convert ambiguous units.
- Do not present unsupported analysis options as selectable.
- Preserve form input following recoverable failures.
- Warn about leaving a dirty form when practical.
- Summarize the request before submission when this reduces mistakes.
- Do not claim requested data exists until the backend confirms availability.
- Avoid silently defaulting scientifically significant parameters.

The exact set of target types, analysis toggles and required inputs must come from the actual capability response and API schema, not from assumptions in the UI.

For Public Explorer coordinates and FOV:

- Display RA and Dec units and the accepted coordinate format next to the inputs.
- Validate finite RA/Dec values and supported ranges before sending the request; backend validation remains authoritative.
- Do not silently convert ambiguous sexagesimal text, infer a coordinate frame, or change coordinate precision without showing the normalized result.
- Display the FOV shape, width/height, and angular units; enforce capability-provided bounds without silently clamping.
- Keep observation-time filters separate from retrieval timestamps and product release dates.
- Show spectral channel, broad band, and detector array as separate optional filter groups when supported; never label them as time controls.
- A target-name resolver, if enabled, must display ambiguity and source provenance and must not silently choose a component.

---

## 14. Accessibility, Responsiveness and Interaction Quality

### Accessibility
- Use semantic HTML and correct heading hierarchy.
- Every input has an associated label.
- All navigation and actions work with keyboard.
- Focus is visible and logical, including in dark and light themes.
- Dialogs manage focus and return it to the trigger when closed.
- Do not use color as the only status signal.
- Buttons have accessible names; icon-only controls need labels/tooltips.
- Tables have meaningful headers.
- Error messages are perceivable by assistive technology.
- Respect reduced-motion preferences.
- Avoid auto-playing disruptive animations.

### Public Explorer viewer accessibility

- Play/pause, previous/next, timeline scrubbing, sequence selection, zoom, and metadata controls must be keyboard-accessible with visible focus.
- Expose the selected frame index and actual observation timestamp to assistive technology.
- Do not autoplay on initial page load. Respect reduced-motion preferences and provide a non-animated way to inspect frames.
- Do not rely on color, blinking speed, or animation alone to communicate a data-quality or scientific status.
- Keep a readable text/metadata alternative available when an image artifact cannot be loaded.

### Responsive behavior
- Test desktop, tablet and narrow mobile layouts.
- Collapse the sidebar without losing accessible labels.
- Stack filters and content regions on small screens.
- Ensure buttons and inputs remain easy to operate.
- Let wide technical tables scroll within a bounded region or provide a mobile-specific view.
- Keep critical statuses and resource identity visible.
- Do not sacrifice scientific labels/units to fit a chart; provide a text/table alternative.

### Motion and polish
- Keep transitions subtle and purposeful.
- Do not animate an execution status unless the backend state changes.
- Do not add fictional “live telemetry”, rotating stars, arbitrary progress, or discovery celebration effects.
- Skeletons are acceptable during real loading; they must disappear when the request resolves.

---

## 15. Security

Follow `SECURITY.md`.

### Application access boundaries

- Public Explorer query/frame/artifact operations may be unauthenticated only as explicitly permitted by the backend public read-only allowlist and rate/resource policy.
- Public access does not grant access to private investigations, user data, reviews, or admin operations.
- The User Console must use the backend's approved authentication/session strategy. Do not invent token storage or rely on hidden navigation as authorization.
- Admin Console access requires backend-enforced administrative roles and explicit audit behavior; do not ship unprotected admin routes.
- No frontend application may connect directly to PostgreSQL, Redis, object storage, IRSA, Gaia, WISE, or LLM providers. All external access goes through approved backend adapters/services.
- Artifact images must use controlled artifact IDs/content endpoints. Do not expose internal storage keys, absolute paths, arbitrary provider URLs, or unrestricted archive proxy endpoints.
- Public query size, FOV, time range, result count, concurrency, and artifact transfer are enforced by the backend. Client-side limits are usability aids only.

- Never ship secrets in the frontend bundle.
- Do not expose archive or LLM provider keys to the browser.
- Do not trust client-side access checks; backend authorization is authoritative.
- Configure only expected API origins and CORS in development/deployment.
- Avoid rendering untrusted data as raw HTML. Sanitize Markdown/HTML report content and use a safe renderer.
- Treat archive/catalogue fields and report references as untrusted text.
- Do not execute or evaluate data returned from the API.
- Do not include credentials, sensitive payloads or stack traces in user-visible errors.
- Validate `VITE_API_BASE_URL` at startup/use and show a safe configuration error if missing.
- Do not store large scientific files or secrets in local storage.
- Store only appropriate preferences such as theme when needed.
- When authentication is introduced, follow the backend's reviewed authentication/session strategy; do not invent a token-storage scheme in the frontend.
- Ensure direct URL/resource access is checked by the backend before deployment to multiple users.

### Markdown and report rendering

If reports contain Markdown:
- Use a maintained Markdown renderer.
- Disable raw HTML or sanitize it with a reviewed sanitizer.
- Do not allow arbitrary scripts, event handlers or untrusted iframe content.
- Render evidence links through controlled navigation logic.
- Show an error for unavailable references rather than silently ignoring them.

---

## 16. Testing

Follow `TESTING_AND_ACCEPTANCE.md`. A test plan does not imply that tests have been implemented or passed. Tests must cover all frontend surfaces in scope and use real backend contracts or clearly labelled fixtures.

### 16.1 Public Explorer component/unit tests

- RA/Dec parsing, formatting, supported coordinate frames, and invalid-range messages.
- FOV bounds, units, supported region shape, and rejection of unsupported values without silent clamping.
- Capability-dependent query filters.
- Query loading, queued/running, terminal, successful-empty, single-observation, partial-result, unsupported, insufficient-data, and failure states.
- Frame-manifest sequence grouping and stable manifest/version references.
- Time ordering uses actual observation timestamps, not retrieval time or spectral channel number.
- Spectral channel, broad-band, detector-array, and epoch dimensions remain distinct.
- Frame viewer play/pause, previous/next, scrubbing, selected timestamp, one-frame state, and zero-frame state.
- Blink/side-by-side controls do not label visual changes as measured motion.
- Metadata panel displays unavailable fields honestly.
- Artifact-image loading, content type/size failures, pending/expired artifacts, and safe content delivery.
- Shareable URL parsing/serialization contains only allowed public state.
- Explicit investigation handoff requires user action and sends stable IDs, not image payloads or client measurements.

### 16.2 User Console component/unit tests

- Status label mapping, including unknown future values.
- Loading/empty/error/partial states.
- Measurement and uncertainty display.
- Capability-dependent form options.
- Form validation and preserved values after errors.
- Confirmation dialog behavior.
- Theme token application.
- Evidence navigation.
- Safe report Markdown rendering.
- Workspace “Open/Resume” does not create a duplicate execution.

### 16.3 API/service tests

- Explorer query accepted response and status polling.
- Observation list and paginated frame-manifest retrieval.
- Manifest version changes and stale response handling.
- Artifact metadata and binary/content response handling.
- API validation error, provider error, and successful empty result remain distinct.
- API client success/error parsing with request ID.
- Pagination and optional fields.
- Correct handling of `null`, omitted fields, and empty arrays.
- Mutation pending state and duplicate-submit guard.
- Polling cleanup on terminal state or unmount.
- Stale/aborted read-response handling.
- Investigation handoff uses `source_context` and receives a server-created `source_snapshot_id`.
- Starting a run is a separate explicit action and includes the source snapshot for Explorer-origin investigations.
- Frontend clients never make direct database, object-store, or external-archive requests.

### 16.4 End-to-end acceptance journeys

**Public Explorer journey:**

1. Open the public Explorer without logging in.
2. Enter valid RA/Dec and a supported FOV.
3. Submit a sky-region query and receive a query ID.
4. Observe the actual query state and coverage outcome.
5. Load observation records and the versioned frame manifest.
6. Select a compatible sequence and inspect the returned frames.
7. Use playback/timeline controls and verify actual observation timestamps are shown.
8. Inspect frame metadata and artifact content.
9. Exercise a zero-result or one-frame state and confirm no synthetic data/frames appear.

**Explorer-to-Investigation journey:**

1. Select a region/sequence/frames in the Explorer.
2. Explicitly choose “Investigate”.
3. Confirm the client sends stable query/manifest/frame/observation references only.
4. Verify the backend-created investigation and immutable source snapshot are shown only after successful response.
5. Verify no execution or agent run starts until the user separately requests it.
6. Start the run through the User Console and confirm the returned `source_snapshot_id` is used.

**User Console investigation journey:**

1. Open Workspace.
2. Create or open an investigation.
3. Start a workflow.
4. Observe real execution/task status from the API.
5. Open available evidence/results.
6. Request report generation when permitted.
7. Open the completed report.
8. Verify provenance and limitations are visible.
9. Exercise at least one failed or insufficient-data path and confirm the UI does not display false success.

### 16.5 NASA challenge target checks

Test the Public Explorer against:

- Barnard's Star.
- Epsilon Indi B, preserving pair/component ambiguity as applicable.
- WISE 0855 using the challenge-supplied coordinate `08h 55m 10.831s`, `−07° 14′ 42.53″`.
- RW Aurigae.
- R Draconis.

For named targets without challenge-supplied coordinates, use the configured authoritative resolver/catalogue and retain identifier, coordinate frame/epoch, and source. Do not invent coordinates. The UI test passes when the actual data availability, returned observations, limitations, and ambiguity are represented accurately. It does not require positive detection, motion, or variability for every target.

### 16.6 Release blockers

Do not call the frontend platform MVP-ready until:

- The Public Explorer works independently of login and the investigation engine.
- Coordinate query, FOV selection, actual observation discovery, frame manifest, viewer, and timeline work against a supported backend or clearly labelled fixture adapter.
- No fake frames, timestamps, observations, progress, or scientific values are presented.
- Explorer handoff is explicit and snapshot-backed; browsing alone creates no investigation/agent run.
- The User Console investigation-to-report journey works for at least one supported workflow.
- Backend and query/task states, failure, cancellation, blocked, unsupported, insufficient-data, and empty-result outcomes are distinct.
- Artifact delivery does not expose arbitrary URLs, storage paths, or credentials.
- Nested routes work after refresh.
- Responsive layouts, keyboard operation, reduced-motion behavior, and contrast are checked.
- Evidence links and report versions behave correctly.
- Unsupported export formats, comparisons, overlays, or admin capabilities are not shown as available.
- Remaining limitations are documented.

## 17. Implementation Roadmap

Build the required public Explorer path first, then complete the explicit handoff and the minimum investigation-to-report path. Keep the three frontend applications separate from the start. Do not delay P0 functionality for optional comparisons or admin features.

### Phase 0 — Contract and repository review

- Read `PRD.md`, `SYSTEM_ARCHITECTURE.md`, `API_SPECIFICATION.md`, `DATABASE_SCHEMA.md`, `DATA_SOURCE_SPEC.md`, `SCIENTIFIC_METHODOLOGY.md`, `UI_UX_SPECIFICATION.md`, `SECURITY.md`, and `TESTING_AND_ACCEPTANCE.md`.
- Confirm root pnpm workspace structure and separate app folders.
- Confirm the current generated OpenAPI contract, exact query/manifest/artifact schema, public access policy, and capability endpoint.
- Identify which backend features exist versus remain planned.
- Confirm PostgreSQL is backend-only and there is no frontend database connection.
- Resolve contract differences before coding.

**Exit criteria:** app boundaries, routes, types, and endpoint clients are based on the active contracts.

### Phase 1 — Workspace and shared foundations

- Configure the root workspace and the three Vite/React/TypeScript apps as separately buildable applications.
- Establish design tokens, typography, spacing, focus styles, and accessible primitives.
- Create the Public Explorer app shell first; keep User Console and Admin Console roots separate.
- Establish generated API types or contract checks and a shared typed API client strategy.

**Exit criteria:** apps build independently; shared UI/contracts do not blur feature or access boundaries.

### Phase 2 — Public Explorer search and query integration

- Implement coordinate input and parsing supported by the API.
- Implement capability-driven FOV and optional filters.
- Submit `POST /explorer/queries` and track the actual asynchronous query state.
- Implement truthful coverage, empty, partial, unsupported, insufficient-data, and failure states.

**Exit criteria:** a user can submit a bounded sky-region query and see its actual backend outcome without logging in or starting an investigation.

### Phase 3 — Observation manifest, artifact delivery, and viewer

- Retrieve observation records and the versioned frame manifest.
- Implement compatible sequence selection and actual-time frame ordering.
- Implement artifact-backed frame display, metadata, timeline, playback, and basic visual blinking.
- Handle zero/one-frame coverage without creating synthetic frames.
- Add URL-addressable public query state when the contract supports it.

**Exit criteria:** the NASA P0 coordinate-to-observation-to-image-frame journey works end to end.

### Phase 4 — Explicit handoff and User Console foundation

- Implement the `source_context` handoff from the Explorer through `POST /investigations`.
- Display the returned immutable source snapshot reference only after successful creation.
- Keep run creation as a separate explicit action.
- Implement User Console shell, Workspace, and New Investigation using the exact API capability/request schemas.

**Exit criteria:** Explorer browsing remains independent; an explicit selected-source handoff creates a reproducible investigation.

### Phase 5 — User Console execution lifecycle

- Implement execution/task monitor using real API state.
- Add polling cleanup, stale-response handling, cancellation, and retry only where supported.
- Implement evidence/measurement/candidate views for result types the backend actually supports.

**Exit criteria:** an authorized user can monitor one supported investigation without fake progress or duplicate actions.

### Phase 6 — Reports, capabilities, and optional features

- Implement report rendering, version display, limitations, and evidence navigation.
- Implement capability display from the backend.
- Add optional image comparisons, spectra, or Gaia/WISE overlays only after the backend capability and methods are validated.
- Add Admin Console functionality only for approved operational requirements and protected endpoints.

**Exit criteria:** reports and advertised capabilities are traceable to backend data; optional controls are capability-gated.

### Phase 7 — QA and release

- Run unit/component/API integration and end-to-end tests for the Explorer and User Console.
- Test five NASA challenge targets and coverage/ambiguity states.
- Test direct navigation, refresh, mobile layout, keyboard, reduced motion, and contrast.
- Verify no frontend secrets, direct storage/DB access, fake science, or false discovery labels.
- Reconcile docs with what actually works.

**Exit criteria:** frontend acceptance criteria pass; remaining limitations are documented.

## 18. Definition of Done

A frontend feature is complete only when:

- [ ] It is implemented in the correct application root (`frontend/`, `user-console/`, or `admin-console/`).
- [ ] It matches the source-of-truth product, UI, API, security, and scientific contracts.
- [ ] It uses typed API data or explicitly labelled fixture data.
- [ ] Loading, empty, error, partial, unsupported, and insufficient-data states are handled where applicable.
- [ ] Duplicate actions, stale requests, polling cleanup, and route refresh are handled safely.
- [ ] Values, units, observation timestamps, uncertainty, quality flags, and provenance are not fabricated or hidden.
- [ ] Explorer frame playback uses only actual frames from a compatible manifest sequence and is ordered by actual observation time.
- [ ] Explorer browsing does not create an investigation or agent run.
- [ ] Explorer-origin investigations use a server-validated immutable source snapshot.
- [ ] The frontend never connects directly to PostgreSQL, Redis, external archives, or object storage.
- [ ] Artifact delivery uses controlled backend endpoints and does not expose credentials or internal paths.
- [ ] Keyboard, responsive, reduced-motion, and contrast behavior are checked.
- [ ] Tests cover normal and error cases, including at least one genuine empty/insufficient-data state.
- [ ] Any new API requirement is coordinated with the backend owner.
- [ ] Setup and developer instructions are updated.
- [ ] No secret or private key is bundled into the frontend.
- [ ] A feature is documented as implemented only after it works and its tests pass.

## 19. Non-negotiable Frontend Rules

1. **Correct application boundary.** `frontend/` is the Public Explorer; `user-console/` is the authenticated investigation workspace; `admin-console/` is restricted operations.
2. **Explorer independence.** Public sky search and image playback must work without creating an investigation or invoking an agent.
3. **Backend is the source of truth.** Never invent persistent query, observation, frame, task, candidate, measurement, or report state.
4. **No fake time series.** Use actual returned frames and observation timestamps. No synthetic epochs or fabricated coverage.
5. **No dimension confusion.** Spectral channels, broad bands, detector arrays, observation epochs, release, and retrieval time are separate dimensions.
6. **No fake science.** Never invent measurements, uncertainties, observations, catalogue matches, or physical interpretations.
7. **No false discovery.** Candidate prioritization or visual movement is not a confirmed planet.
8. **No hidden uncertainty.** Show quality outcomes, missing data, limitations, and contradictory evidence.
9. **No silent unsupported controls.** Hide or clearly disable features that are not implemented and validated.
10. **Explicit investigation handoff.** Pass stable query/manifest/frame/observation references; let the backend validate and snapshot them.
11. **Separate investigation start.** Creating an investigation/source snapshot does not automatically start a workflow or agent run.
12. **No direct database or storage access.** Frontend apps use the FastAPI API and controlled artifact endpoints only; PostgreSQL remains backend-only.
13. **No secret frontend configuration.** Anything bundled into the browser is public.
14. **Safe artifact rendering.** Never render arbitrary provider URLs, raw filesystem paths, or unsanitized report HTML.
15. **No broken deep links.** Routes must work on direct load and browser refresh.
16. **MVP first.** Deliver a reliable P0 Explorer and a bounded investigation-to-report path before advanced visualization, collaboration, or admin expansion.
