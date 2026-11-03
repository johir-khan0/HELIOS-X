
# HELIOS-X Security

**Project:** HELIOS-X — Hierarchical Exploration & Longitudinal Investigation of Objects  
**Document:** `SECURITY.md`  
**Version:** 1.1 (MVP — architecture-aligned)  
**Status:** Updated security requirements

## 1. Purpose and Scope

This document defines the minimum security rules for developing and running the HELIOS-X MVP. It covers secrets, access control, input validation, agent/tool permissions, untrusted scientific data, logs, dependencies, artifact handling, external data retrieval, and protection of sensitive information.

HELIOS-X has two connected but independently usable capabilities:

- **Public SPHEREx Sky Image Explorer:** anonymous, bounded, read-only access to designated public sky-query, frame, metadata, and image-delivery operations.
- **Agentic Scientific Investigation Engine:** explicitly initiated investigations, private user resources, evidence, candidate assessments, reports, and human review.

The Public Explorer must remain usable without an account, investigation, or agent execution. Private investigation resources and administrative operations require backend-enforced authorization. The User Console and Admin Console are separate frontend applications; separating their interfaces does not replace backend access control.

The MVP should use simple, enforceable controls. More advanced infrastructure should be added only when a real deployment requirement justifies it. This document does not claim that the controls have already been implemented or audited.

## 2. Security Principles

- **Least privilege:** each component receives only the access needed for its task.
- **Validate at boundaries:** validate API requests, configuration, files, archive responses, artifacts, and tool results before use.
- **Treat external content as data:** data files, catalogue records, metadata, filenames, and retrieved text must never be treated as trusted instructions.
- **Keep secrets out of code and logs.**
- **Separate public and private capabilities:** anonymous Explorer access must not expose private investigations, reviewer actions, administrative operations, or arbitrary server files.
- **Make actions traceable:** record security-relevant events without recording sensitive payloads unnecessarily.
- **Fail safely:** invalid input, unsupported operations, and permission failures must not silently succeed.
- **Scientific integrity:** agents must not invent measurements, skip failed quality gates, or overwrite evidence to make a workflow appear successful.
- **No trust by identifier alone:** UUIDs, artifact IDs, frame IDs, and opaque references are not authorization credentials.

## 3. Secrets and Configuration

### Requirements

- Never commit API keys, passwords, access tokens, private keys, or production credentials to Git.
- Commit a `.env.example` containing placeholder values only; ignore real `.env` files in Git.
- Read secrets from environment variables or a deployment-provided secret store.
- Do not place secrets in frontend source code. Anything shipped to a browser must be treated as public.
- Do not expose secrets in API responses, exception messages, screenshots, reports, or logs.
- Use separate credentials for development and any deployed environment.
- Rotate any credential that is accidentally committed or exposed; deleting it from the latest commit alone is not enough.
- Validate required configuration at startup and fail with a clear, non-secret error message.
- Keep archive, catalogue, LLM-provider, artifact-storage, and database credentials on the backend. Do not send them to an agent as raw credentials when a narrowly scoped backend service can perform the operation.

For local development, a git-ignored `.env` file is acceptable. A managed secret store may be used when deployed; it is not required for the local MVP.

## 4. Authentication and Authorization

### 4.1 Local MVP

If the API is bound to localhost and used by one trusted developer, a full identity system may be out of scope. The application must still avoid exposing the development server to untrusted networks by default. Binding the API to a LAN or public interface changes the exposure and requires the applicable authentication, authorization, and network controls before access is enabled.

### 4.2 Access boundaries

The application has three separately buildable client surfaces:

- **Public Explorer (`frontend/`):** anonymous access is limited to explicitly designated read-only public operations and public artifacts. Browsing, coordinate lookup, frame inspection, playback, blinking, and comparison must not create an investigation, candidate, or agent run. An explicit user action is required to hand selected Explorer context to the investigation workflow.
- **User Console (`user-console/`):** authenticated user workflows and private investigation resources. A user may access only resources they own or are otherwise authorized to access.
- **Admin Console (`admin-console/`):** restricted operational interface. Administrative functionality is conditional on an approved operational requirement and must remain unavailable to public users. Do not create or expose placeholder administrative endpoints that have no approved backend behavior.

### 4.3 Resource access rules

Before shared or public deployment:

- Require authentication for private application resources.
- Enforce authorization on the backend; hiding buttons in the frontend is not access control.
- Enforce ownership/role checks on every request involving an investigation, execution, task, evidence item, candidate, report, selection snapshot, review decision, retry, or cancellation.
- Do not trust client-supplied owner IDs, role names, reviewer assignments, or authorization flags. Derive the effective identity and permissions from the authenticated server-side context.
- A reviewer may access and review only resources assigned or otherwise explicitly authorized to that reviewer.
- Administrative access must use explicit administrator permissions. Administrative status does not automatically authorize changing scientific measurements, evidence, or quality-gate outcomes.
- Restrict administrative and operational endpoints independently of whether the Admin Console frontend is deployed or discovered.
- Configure explicit CORS origins. Do not use an unrestricted wildcard with credentialed requests.
- Use HTTPS through the deployment environment.
- Apply rate limits and request-size limits to public endpoints.

The following access matrix defines the intended boundary. It is not proof that identity infrastructure or every role has already been implemented.

| Resource / action | Public visitor | Authenticated user | Reviewer | Administrator |
|---|---|---|---|---|
| Public sky query and capability metadata | Allow, bounded and rate-limited | Allow | Allow | Allow |
| Public image frame/artifact | Allow through controlled delivery | Allow | Allow | Allow |
| Create private investigation | Deny unless a separately approved guest flow exists | Own resources | Only when explicitly authorized | Only where operational policy permits |
| Read private investigation and related resources | Deny | Own/authorized resources | Assigned/authorized resources | Only with explicit, audited permission |
| Cancel or retry execution | Deny | Own/authorized resources | Only if assigned role permits | Restricted operational policy |
| Record candidate review | Deny | Only with explicit reviewer permission | Assigned resources | Only with explicit reviewer/admin policy |
| Change application/source configuration | Deny | Deny | Deny | Allow through validated, audited actions |

Do not claim multi-user isolation or role support until the corresponding backend controls and tests exist.

### 4.4 Session and request protection

- Use a reviewed authentication/session mechanism appropriate to the deployment; this document does not select an identity provider.
- Protect session credentials against exposure and unauthorized reuse. Apply expiry and revocation behavior appropriate to the selected mechanism.
- If browser authentication uses cookies, set appropriate `Secure`, `HttpOnly`, and `SameSite` attributes and implement CSRF protection for state-changing requests.
- Do not place long-lived access tokens or privileged credentials in browser local storage or frontend bundles.
- Require strong authentication for administrative access; enable multi-factor authentication where supported by the selected deployment identity system.

## 5. Input Validation, External Retrieval, and File Handling

### 5.1 API and scientific input validation

- Validate all API request bodies with Pydantic schemas.
- Validate query parameters, IDs, enum values, coordinates, time ranges, pagination limits, and sort fields.
- Reject non-finite numeric values where scientific calculations expect finite numbers.
- Enforce supported coordinate ranges, supported FOV values, date/time constraints, channel selections, result-count limits, and other capability limits on the backend.
- Do not rely on frontend validation for security or resource control.
- Apply configurable limits to request bodies, query complexity, result counts, concurrent requests, processing time, and returned bytes.

### 5.2 Public Explorer limits

Anonymous Explorer access is intentionally limited. The backend must enforce configured limits for:

- RA/Dec and supported field-of-view values;
- time ranges and supported spectral-channel selections;
- query result count and frame count;
- cutout/preview dimensions and output size;
- archive request timeouts, download size, processing time, and concurrency;
- request frequency and repeated/abusive query patterns.

The Public Explorer may use designated read-only query, capability, frame-manifest, and controlled image-delivery operations. An Explorer query may create bounded query metadata or derived display artifacts when defined by the API contract, but it must not create an investigation, candidate, or agent run. Playback and frame navigation must not invoke the agent runtime.

### 5.3 External archive and catalogue retrieval

- Restrict server-side retrieval to configured, approved providers and supported archive/catalogue hosts, including the approved IRSA integration.
- Do not accept an arbitrary user-supplied URL as a source to fetch from the backend.
- Do not expose an unrestricted proxy to IRSA, another archive, or arbitrary network destinations.
- Validate provider identifiers, product references, and retrieval parameters before making an outbound request.
- Validate redirect destinations as well as the initial destination. Reject redirects to unapproved hosts or disallowed address ranges.
- Protect against SSRF, including requests to loopback, private, link-local, multicast, and other internal/reserved network destinations. Apply appropriate DNS-resolution and connection-time checks to reduce DNS-rebinding risk.
- Apply outbound connection timeouts, response-size limits, and bounded retries. Do not allow an external response to consume unbounded memory, disk, or processing time.
- Keep provider credentials on the backend and use the narrowest available credentials and permissions.

### 5.4 File and scientific-product handling

- Apply size limits to requests and uploaded or downloaded files.
- Validate file type and expected structure; do not trust a filename or extension alone.
- Treat FITS headers, catalogue values, metadata, filenames, archive responses, and report source text as untrusted input.
- Validate FITS and other scientific products before processing, including relevant header structure, HDU count, dimensions, data type, pixel count, and compressed/decompressed size.
- Apply configurable limits for parser time, memory, decoded dimensions, decompression expansion, and concurrent file processing. Reject products that exceed supported limits.
- Use safe parsing libraries and supported code paths. Never execute content, macros, scripts, or instructions embedded in a data file or metadata field.
- Do not accept arbitrary filesystem paths, shell commands, Python code, SQL fragments, or executable content from the user.
- Use parameterized database operations through the persistence layer; never build SQL by concatenating untrusted input.
- Store retrieved files in controlled application-managed storage, not an arbitrary path supplied by a client.
- Prevent path traversal and symlink-based access outside approved storage locations. Resolve and validate the final storage target before reading or writing.
- Where checksums are available or computed, record and verify them.
- Do not process malformed files as if they were valid observations.

## 6. Artifact Access and Storage Security

Large FITS files, image arrays, spectra, previews, cutouts, and report binaries belong in controlled artifact storage, not in PostgreSQL rows or ordinary JSON response bodies. PostgreSQL stores the relevant artifact metadata and references, not the large binary content.

- Store artifacts outside the public web root and outside source control.
- Expose artifacts only through controlled artifact IDs and approved content-delivery operations; never return absolute server filesystem paths.
- An artifact ID or opaque reference is not proof of permission. Check artifact visibility and the caller's authorization on every metadata and content request.
- Mark public artifacts explicitly. Anonymous users may access only artifacts designated public by backend policy; private investigation artifacts and reports require resource-level authorization.
- Verify that an artifact belongs to the expected source product, query, investigation, execution, or report before delivery.
- Prevent path traversal, symlink escapes, arbitrary file reads, arbitrary remote URL fetches, and unbounded downloads.
- Validate stored media type and content metadata. Do not trust a client-provided content type or filename.
- Use safe response headers and avoid rendering untrusted HTML or active content in the browser. Render Markdown or report content using a safe, non-executable rendering policy.
- If short-lived signed access references are used by the storage implementation, scope them to the intended artifact and operation, use a limited expiry, and do not expose storage credentials.
- Apply appropriate size, caching, expiry, and download limits without breaking source-product provenance.
- Preserve the relationship between derived previews/cutouts and their original source product and processing configuration.

## 7. Agent and Tool Access

HELIOS-X contains eight logical agent roles. Agents should use approved interfaces exposed by the application, not unrestricted operating-system or infrastructure access.

- Give each role only the tools and evidence needed for its defined task and authorized investigation.
- Prefer typed function calls and allowlisted operations.
- Do not allow agents to execute arbitrary shell commands, arbitrary Python code, unrestricted SQL, direct ORM/database queries, arbitrary filesystem operations, or arbitrary network requests.
- Agents must not receive PostgreSQL credentials, artifact-storage credentials, provider secrets, or administrator credentials.
- Restrict data retrieval to configured/approved providers and supported products through backend services.
- Enforce role-specific tool permissions and investigation-level evidence access. An agent must not read data from another user's or another investigation's private context.
- Do not let an agent change permissions, secrets, runtime configuration, access-control policy, or its own tool allowlist.
- Validate tool arguments before execution and validate returned data before downstream use.
- Enforce timeouts, bounded retries, and resource limits for tool calls.
- Record tool name, task ID, outcome, and relevant non-sensitive metadata.
- Do not expose hidden chain-of-thought or internal prompts in user-facing output. Store concise evidence-linked rationale and structured outputs where required, not private internal reasoning.
- The Workflow Orchestrator and deterministic scientific services enforce state transitions and quality gates; agents cannot override them through narrative output.
- Ordinary Explorer browsing, coordinate lookup, image playback, blinking, and frame inspection must not start an investigation or agent run. An explicit, validated user handoff is required.

## 8. Untrusted Data and Prompt-Injection Resistance

External files, catalogue records, provider responses, and retrieved text may contain misleading text or instruction-like content.

- Treat all retrieved content as data, never as instructions that override system rules.
- Separate trusted application instructions from untrusted data when passing content to an LLM.
- Do not follow instructions embedded in a data record to reveal secrets, call unapproved tools, modify system behavior, or expand permissions.
- Do not allow external content to expand an agent's tool permissions or alter the Workflow Orchestrator's state.
- Do not place secrets, privileged credentials, or unnecessary private user data in model context.
- Use bounded, relevant excerpts rather than passing entire files when unnecessary.
- Validate claims against structured tool outputs and evidence references.
- Test with fixture data containing instruction-like strings and verify that agents ignore them as commands.

## 9. Scientific Results and Data Integrity

- Scientific measurements must come from deterministic, tested computation, not fabricated LLM output.
- Preserve the source reference, method/version, units, uncertainty where available, quality flags, and task provenance.
- Preserve source observation/product identifiers, release information, actual observation timestamps, retrieval context, and artifact references where available.
- For an Explorer-originated investigation, persist the validated selection snapshot and revalidate it and the current data/capability state before execution. Do not rely only on transient frontend state or unverified client-supplied frame references.
- Do not silently rewrite previous task results or assessments; preserve version/attempt history where required by the application.
- A failed quality gate must remain visible to downstream tasks and reports.
- A missing observation, unavailable product, failed retrieval, or catalogue no-match must not be represented as a positive scientific finding or proof of a new object.
- SPHEREx spectral channels must not be treated as separate time epochs. Temporal playback must use actual distinct observation timestamps.
- Do not fabricate missing frames or imply that every sky region has observations at a fixed six-month interval.
- Sample/demo data must be clearly labelled and must not be represented as real detections.
- Human review decisions must be recorded with the reviewer identity, time, rationale, and the evidence/result version reviewed, subject to the approved privacy policy. A review decision must not silently alter source observations or computed measurements.

## 10. Database and Persistence Security

PostgreSQL is the selected relational system of record for local development, CI/integration tests, staging, and production. SQLite is not the application database or an automatic fallback. Database-platform details belong to `DATABASE_SCHEMA.md` and `OPERATIONS.md`; this section defines the security requirements.

- Keep the PostgreSQL connection URL and credentials in environment/secret configuration, never in source control, frontend code, reports, logs, or ordinary business records.
- Use separate least-privilege PostgreSQL migration-owner and application-runtime roles.
- The runtime role must not be a superuser, schema owner, or unrestricted DDL principal. Grant only the permissions needed by the application.
- Restrict database network exposure to the application and approved operators. Do not expose PostgreSQL directly to public clients or frontend applications.
- Use TLS for remote database connections over untrusted networks.
- Agents and frontend clients must not connect directly to PostgreSQL. Database access remains within the backend persistence layer.
- Enforce ownership and authorization in backend service queries; UUIDs are not authorization. Do not assume database row-level security is active unless it is explicitly configured and tested.
- Keep transactions bounded and avoid holding database transactions open during archive calls, artifact transfers, or LLM calls.
- Do not store provider credentials or secrets in database records.
- Keep large scientific binaries in controlled artifact storage rather than relational rows.
- Protect migrations and schema-changing operations. Run them through the approved deployment/migration process, not through agent tools or public API requests.

## 11. Logging, Audit Events, Errors, and CORS

### 11.1 Logging and audit events

Record enough to investigate failures and security-relevant actions, including:

- Timestamp and request/correlation ID.
- Route or operation name and response status.
- Investigation, execution, task, frame, and artifact IDs when applicable and safe to record.
- Authentication failures, authorization denials, and repeated public-endpoint abuse.
- Run, cancellation, retry, report-generation, review-decision, and configuration-change events.
- Artifact metadata/content access events where needed for security auditing.
- Tool name, outcome, and method/version where relevant.
- Administrative role, permission, or configuration changes, including the acting identity and result.

Do not log:

- Passwords, tokens, API keys, secret environment values, or database connection strings.
- Full authorization headers or session credentials.
- Unnecessary personal data or private investigation content.
- Large raw scientific files, image arrays, or complete unfiltered provider responses.
- Hidden prompts or private internal reasoning.
- Unfiltered untrusted strings that could forge or corrupt log entries.

Use structured logs and a configurable log level. Sanitize untrusted log fields, protect log access, and apply a documented retention policy. Error responses to clients must not expose stack traces, credentials, SQL text, internal filesystem paths, or private resource details.

### 11.2 Error handling and CORS

- Return stable, documented error codes and readable messages.
- Do not return stack traces or secret configuration to clients.
- Avoid revealing whether a private resource exists when the caller is not authorized to access it.
- Validate `Origin` and configure CORS for the expected frontend origins only.
- Avoid broad cross-origin access in a deployed environment.
- Limit request body and file sizes.
- Ensure a failed request does not leave a workflow or database record appearing successful.

## 12. Backup, Retention, and Sensitive Files

- Protect the PostgreSQL database, configuration, downloaded observations, generated artifacts, and reports according to their sensitivity.
- Keep backups outside the public web root and do not commit them to Git.
- Encrypt backups where supported by the selected deployment/storage environment and restrict backup access to authorized operators.
- Back up relational records and the referenced artifacts together, or maintain a verifiable manifest and recovery procedure that ensures the database does not point to missing required artifacts after restoration.
- Document PostgreSQL backup/point-in-time recovery, restore access, backup retention, and deletion rules in `OPERATIONS.md` before deployment.
- Test restoration into a separate environment; a backup that has never been restored is not a verified recovery process.
- Restrict filesystem and artifact-storage access to the application service and intended operators.
- Document where downloaded data, cache, temporary files, and generated reports are stored and how they are removed.
- Do not delete source data or evidence needed to audit a result without an explicit retention rule.
- Apply documented retention and deletion rules to user investigations, logs, cache, and generated artifacts. Retain only the data needed for the defined purpose and audit requirements.
- Before sharing a report externally, check whether it contains local paths, secrets, private notes, personal data, or unreviewed claims.

## 13. Dependency and Source-Code Security

- Pin or lock project dependencies for repeatable builds.
- Prefer maintained packages from trusted package registries.
- Review newly added dependencies and their permissions/licensing where relevant.
- Run the project's available dependency vulnerability checks periodically and before a release.
- Do not install packages dynamically in a running application.
- Keep Python and frontend dependencies reasonably current, applying security fixes when needed.
- Use code review for changes affecting authentication, file handling, external retrieval, artifact access, tool permissions, data persistence, or scientific result integrity.
- Keep `.env`, PostgreSQL dumps, databases containing non-public data, downloaded data, private artifacts, build outputs, and logs out of source control unless explicitly intended and reviewed.

## 14. Minimum Security Acceptance Checklist

Before accepting the MVP:

- [ ] Real secrets are absent from the repository and frontend bundles.
- [ ] `.env`, PostgreSQL dumps, private database/data files, downloaded data, artifacts, and logs are ignored by Git as appropriate.
- [ ] Public Explorer operations are explicitly allowlisted, read-only, and backend-limited.
- [ ] Explorer browsing and playback do not create investigations, candidates, or agent runs; an explicit handoff is required.
- [ ] Private investigations and related resources require backend authentication and resource-level authorization.
- [ ] Reviewer access is limited to assigned or explicitly authorized resources.
- [ ] Administrative endpoints are restricted, role-checked, and audited; unimplemented admin functions are not exposed as working controls.
- [ ] API request schemas, coordinate/time/FOV inputs, provider responses, and file inputs are validated.
- [ ] Public query, request, download, parser, and processing resource limits are enforced.
- [ ] No endpoint accepts arbitrary commands, SQL, code, URLs, or filesystem paths.
- [ ] External retrieval is restricted to approved providers and protected against SSRF and unsafe redirects.
- [ ] FITS and other scientific products are validated and processed within configured resource limits.
- [ ] Artifact metadata and content access are authorized on every request; public and private artifacts are separated.
- [ ] Agent tools are allowlisted and constrained by role and investigation; agents have no direct database, secret, unrestricted filesystem, or arbitrary network access.
- [ ] External metadata and retrieved content are treated as untrusted data.
- [ ] API errors do not leak stack traces, private resource details, internal paths, SQL, or secrets.
- [ ] Logs omit credentials and unnecessary sensitive payloads and include required security audit events.
- [ ] CORS is explicitly configured for expected origins.
- [ ] PostgreSQL uses least-privilege runtime and migration roles and is not directly exposed to public clients.
- [ ] Database and downloaded/private artifacts are stored outside the public web root.
- [ ] PostgreSQL and referenced-artifact backup/restore procedures are documented and restore-tested before deployment.
- [ ] If deployed beyond localhost, HTTPS, authentication, authorization, session protections, and access-control tests are in place.
- [ ] Scientific provenance, actual observation timestamps, and quality failures remain visible.

---

**End of document.**
