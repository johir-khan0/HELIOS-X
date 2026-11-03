# HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects

HELIOS-X is an astronomical data exploration and scientific investigation platform focused on SPHEREx (Spectro-Photometer for the History of the Universe, Epoch of Reionization and Ices Explorer) observations, utilizing NASA/IPAC Infrared Science Archive (IRSA) products as its primary source.

The platform provides two connected, independently usable core capabilities:
1. **Public SPHEREx Sky Image Explorer:** An open, unauthenticated web experience where anyone can enter celestial coordinates (RA/Dec), select a field of view (FOV), discover actual SPHEREx observations across observation epochs, and navigate temporal image frames with interactive playback, blinking, and metadata inspection.
2. **Agentic Scientific Investigation Engine:** An explicitly initiated, private scientific workflow system coordinating deterministic scientific computations, eight specialized AI agent roles, evidence ledgering with provenance, candidate assessment, and traceable scientific reports.

---

## Architecture Overview

HELIOS-X is structured as a documentation-driven modular platform:

```text
helios-x/
├── frontend/               # Public SPHEREx Sky Image Explorer (React + Vite)
├── user-console/           # Authenticated Investigation Workspace (React + Vite)
├── admin-console/          # Restricted Operational Interface (React + Vite)
├── backend/                # Modular Monolith Application (FastAPI + SQLAlchemy 2.x + Alembic)
├── agents/                 # Eight Logical AI-Agent Roles (helios_x_agents)
├── database/               # Canonical Database Schema Specification & Assets
├── shared/                 # Cross-boundary JSON schemas and contracts
├── tests/                  # Cross-module system, scientific, and acceptance tests
├── docs/                   # Architectural Decision Records (ADRs)
├── scripts/                # Developer setup and operational automation scripts
├── infra/                  # Infrastructure, Docker, and CI configurations
├── data/                   # Controlled runtime scientific products & cutouts
├── artifacts/              # Generated reports & analytical exports
└── logs/                   # Runtime execution logs
```

### 1. Three Dedicated Frontend Surfaces
- **`frontend/` (Public Explorer):** Dedicated to coordinate queries, observation timelines, image playback, and frame inspection. Operates without requiring an account or initiating an investigation.
- **`user-console/` (User Workspace):** Authenticated workspace for managing scientific investigations, execution monitors, evidence ledgers, candidate prioritization, and reports.
- **`admin-console/` (Admin Interface):** Role-protected operational dashboard for system health, data-source registries, job management, and audit logs.

### 2. Backend Modular Monolith (`backend/`)
- Built with **Python 3.11+**, **FastAPI**, and **SQLAlchemy 2.x**.
- **PostgreSQL** serves as the canonical relational database of record.
- **Alembic** manages database migrations under `backend/migrations/`.
- Strict separation between HTTP routing, domain services, deterministic scientific algorithms, data-source adapters, and persistence repositories.
- Controlled filesystem artifact storage (`data/`, `artifacts/`) decouples large FITS/image binaries from relational database rows.

### 3. Eight Specialized AI Agent Roles (`agents/`)
Implemented as logical roles invoked in-process by the Workflow Orchestrator:
1. **Investigation Planner Agent:** Proposes structured investigation task graphs.
2. **Data Retrieval Agent:** Coordinates archive discovery and retrieval requests.
3. **Data Quality Agent:** Inspects FITS quality flags, masks, and validity bounds.
4. **Change Detection Agent:** Analyzes flux variations and temporal differences.
5. **Motion Analysis Agent:** Evaluates astrometric displacement and proper motion.
6. **Spectral & Photometry Agent:** Interprets spectral energy distributions and color indices.
7. **Catalogue Matching Agent:** Matches detections against Gaia, WISE, and astronomical catalogues.
8. **Validation & Hypothesis Agent:** Assesses alternative hypotheses and evaluates evidence gates.

*Note: Deterministic scientific calculations and numerical measurements are performed by deterministic computational modules in the backend scientific engine, never fabricated by AI models.*

---

## Canonical Project Documentation

HELIOS-X is strictly documentation-driven. All architectural boundaries and contracts are governed by authoritative specifications:

- `MASTER_DEVELOPMENT_PROMPT.md` — Development instructions and document review hierarchy.
- `PRD.md` — Product requirements, user personas, MVP scope, and non-goals.
- `SYSTEM_ARCHITECTURE.md` — System architecture, communication boundaries, and trust model.
- `IMPLEMENTATION_PLAN.md` — Phase-by-phase implementation sequence and development milestones.
- `database/DATABASE_SCHEMA.md` — Canonical PostgreSQL database schema, entity models, and provenance rules.
- `API_SPECIFICATION.md` — Canonical REST API endpoints, request/response models, and status values.
- `BACKEND.md` — Backend architecture, module layout, and persistence boundaries.
- `FRONTEND.md` — Frontend design system, component hierarchy, and API alignment.
- `UI_UX_SPECIFICATION.md` — User interface screens, interaction states, and accessibility rules.
- `AGENT_SPECIFICATIONS.md` — Input/output schemas, approved tools, and constraints for the 8 agent roles.
- `SCIENTIFIC_METHODOLOGY.md` — Deterministic mathematical methods, quality gates, and physical bounds.
- `DATA_SOURCE_SPEC.md` — SPHEREx data products, IRSA retrieval routes, and metadata formats.
- `INVESTIGATION_WORKFLOW.md` — Investigation lifecycle, state machine, and dependency DAGs.
- `TESTING_AND_ACCEPTANCE.md` — Test matrix, synthetic fixtures, tolerances, and acceptance gates.
- `SECURITY.md` — Access control boundaries, authentication, and secret protection policies.
- `OPERATIONS.md` — Environment setup, deployment, operational routines, and debugging guide.

---

## Development Prerequisites

- **Python:** 3.11+
- **Node.js:** 20+ LTS & **pnpm** (or npm)
- **Database:** PostgreSQL 16+
- **Git**

Copy `.env.example` to `.env` to configure local settings:
```bash
cp .env.example .env
```

---

## License

This project is licensed under the terms specified in the `LICENSE` file.
