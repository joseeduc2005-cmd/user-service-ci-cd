<!--
SYNC IMPACT REPORT
==================
Version change: (template/uninitialized) → 1.0.0
Bump rationale: Initial ratification — template placeholders replaced with concrete
principles for the first time. MAJOR per semver because this establishes the governance
baseline (no prior version to be backward-compatible with).

Modified principles:
  - [PRINCIPLE_1_NAME] → I. Layered Architecture (N-Tier Separation) (NON-NEGOTIABLE)
  - [PRINCIPLE_2_NAME] → II. Test Discipline: Unit and Functional (NON-NEGOTIABLE)
  - [PRINCIPLE_3_NAME] → III. API Versioning (NON-NEGOTIABLE)
  - [PRINCIPLE_4_NAME] → IV. Centralized Observability via AOP (NON-NEGOTIABLE)
  - [PRINCIPLE_5_NAME] → V. SOLID, DRY, YAGNI Discipline (NON-NEGOTIABLE)

Added sections:
  - Architectural Constraints (replaces [SECTION_2_NAME])
  - Development Workflow & Quality Gates (replaces [SECTION_3_NAME])
  - Governance (filled)

Removed sections: none

Templates requiring updates:
  - ✅ .specify/templates/plan-template.md — "Constitution Check" gate is referential
    ("[Gates determined based on constitution file]") and resolves at /speckit-plan time;
    no edit required.
  - ✅ .specify/templates/spec-template.md — scope/requirement structure is consistent
    with new principles; no edit required.
  - ✅ .specify/templates/tasks-template.md — task categorization already accommodates
    layered structure, tests, logging, and versioning concerns; no edit required.
  - ✅ CLAUDE.md — generic pointer to current plan; no edit required.

Follow-up TODOs: none.
-->

# user-service Constitution

## Core Principles

### I. Layered Architecture (N-Tier Separation) (NON-NEGOTIABLE)

Code MUST be organized into the following layers, each with a single, well-defined
responsibility and a strict dependency direction (higher layers depend only on lower
layers, never the reverse):

- **controller**: HTTP/transport entry points. Translates requests/responses, performs
  input binding, delegates to services. MUST NOT contain business logic, persistence
  calls, or domain rules.
- **service**: Orchestrates business use cases. Coordinates domain objects and
  repositories, enforces transactional boundaries. MUST NOT depend on transport types
  (e.g., `HttpServletRequest`) or persistence framework types directly.
- **domain**: Pure business entities, value objects, and domain rules. MUST NOT depend
  on infrastructure (no framework, persistence, or transport imports).
- **repository**: Persistence abstraction. Owns all data access. MUST expose interfaces
  in terms of domain types; implementations live behind those interfaces.
- **dto**: Data transfer objects for inbound/outbound payloads. MUST be distinct from
  domain entities (no leaking persistence/domain types across the wire).
- **exceptions**: Centralized exception hierarchy plus a single global handler that maps
  exceptions to transport responses. Lower layers throw typed exceptions; controllers
  do NOT contain try/catch chains for business errors.

**Rationale**: Strict separation keeps business logic testable in isolation, prevents
transport/persistence leakage into the domain, and bounds the blast radius of changes.

### II. Test Discipline: Unit and Functional (NON-NEGOTIABLE)

Every feature MUST ship with both:

- **Unit tests** for service and domain logic, executed with all external collaborators
  (repository, transport, third-party clients) replaced by test doubles. Pure, fast,
  deterministic.
- **Functional tests** for each endpoint, exercising the real wired stack
  (controller → service → repository) end-to-end against a real or test-substitute
  data store. Functional tests MUST cover the documented happy path plus the explicit
  error responses defined by the endpoint contract.

A pull request that adds or modifies a service method or an endpoint without the
corresponding unit and functional coverage MUST be rejected at review.

**Rationale**: Unit tests give fast feedback on business rules; functional tests prove
the layers wire together correctly. Neither replaces the other.

### III. API Versioning (NON-NEGOTIABLE)

Every HTTP endpoint MUST be exposed under an explicit version segment in its path
(e.g., `/api/v1/...`). The following rules apply:

- Breaking changes (removed/renamed fields, changed semantics, changed status codes,
  changed error shapes, removed endpoints, stricter validation) MUST be released under
  a new major version (`v2`, `v3`, …); the previous version MUST keep working until a
  documented deprecation window elapses.
- Additive, backward-compatible changes (new optional fields, new endpoints, looser
  validation) MAY ship within the current version.
- New endpoints MUST NOT be created without a version prefix; "unversioned" routes are
  not permitted in production code.
- The current version, deprecation status, and sunset date for each endpoint MUST be
  discoverable from the API documentation/contract artifact for the feature.

**Rationale**: Versioning is the contract with consumers; ad-hoc breaking changes
silently break clients and are not recoverable after deploy.

### IV. Centralized Observability via AOP (NON-NEGOTIABLE)

Cross-cutting logging — including request entry/exit, method-call tracing for
service/repository layers, exception capture, and timing — MUST be implemented through
aspect-oriented mechanisms (interceptors, aspects, decorators, middleware). Business
logic (controller, service, domain, repository code) MUST NOT contain inline logging
statements for these cross-cutting concerns.

Specifically:

- Service and repository methods MUST be observable (entry/exit, arguments where safe,
  duration, thrown exceptions) without any `log.*(...)` calls inside their bodies.
- Logging configuration (format, levels, correlation ID propagation, sensitive-field
  redaction) MUST live in one place and apply uniformly across the service.
- Domain-meaningful events that are not cross-cutting (e.g., a deliberate audit record
  of a business action) MAY be emitted explicitly but MUST go through a domain event
  or audit abstraction, not raw logger calls scattered through use cases.

**Rationale**: Inline logging pollutes business code, drifts in format, and makes
behavioral changes (e.g., adding a correlation ID) require touching every file.
Centralizing via AOP keeps the business code readable and observability uniform.

### V. SOLID, DRY, YAGNI Discipline (NON-NEGOTIABLE)

All production code MUST honor:

- **SOLID**: single-responsibility classes; open for extension, closed for
  modification; substitutable subtypes; segregated, client-specific interfaces; depend
  on abstractions, not concretions (services depend on repository interfaces, never on
  concrete implementations).
- **DRY**: a piece of knowledge has one authoritative representation. Duplicated
  business logic MUST be extracted; copy-pasted blocks across layers are a review
  blocker. Note: incidental similarity (two unrelated methods that happen to look
  alike) is NOT duplication — do not couple them.
- **YAGNI**: implement only what the current user story requires. Speculative
  abstractions, unused configuration knobs, "future-proof" extension points, and
  scaffolding for hypothetical features MUST be removed before merge. If it is not
  exercised by a test in this PR, it does not ship.

**Rationale**: These three together keep the codebase small, consistent, and amenable
to change. Violations compound: today's "just in case" abstraction is next quarter's
dead code that nobody dares delete.

## Architectural Constraints

- The package/module structure MUST mirror the layer names from Principle I
  (`controller`, `service`, `domain`, `repository`, `dto`, `exceptions`). A class's
  layer MUST be unambiguous from its location.
- Dependencies between layers MUST flow downward only:
  `controller → service → (domain, repository)`. `domain` MUST NOT import from any
  other layer. `repository` implementations MAY import `domain` only.
- DTOs MUST NOT be returned from the service layer to the controller as domain
  entities, and domain entities MUST NOT be accepted as request bodies. Mapping
  happens at the controller boundary.
- All endpoints MUST live under a versioned base path (Principle III). The version
  segment MUST appear in the route, not only in headers, unless an explicit ADR
  documents an alternative scheme for a specific endpoint.
- Cross-cutting logging MUST be wired through aspects/interceptors (Principle IV).
  Adding a `logger.info(...)` to a service or repository method requires explicit
  reviewer sign-off in the PR with a justification, and is allowed only for genuine
  business-audit events that cannot be expressed as a cross-cutting concern.

## Development Workflow & Quality Gates

- Every PR MUST pass: build, unit tests, functional tests, and the project's lint
  configuration. A failing gate blocks merge; the failing test/lint MUST be fixed at
  the root cause, not suppressed.
- Every PR description MUST state which user stories/requirements it implements and
  call out any deviation from the principles above (which then routes through the
  amendment procedure in Governance, not through silent merge).
- Code review MUST verify, at minimum: layer boundaries are intact, both unit and
  functional tests exist for new behavior, endpoints are versioned, no inline
  cross-cutting logging was introduced, and no speculative code was added.
- Refactors that touch shared abstractions (interfaces between layers, the exception
  hierarchy, the logging aspect) MUST be reviewed by at least one maintainer beyond
  the author.

## Governance

- This constitution supersedes any ad-hoc convention, README snippet, or chat
  agreement. When this document and another source disagree, this document wins until
  amended.
- **Amendment procedure**: Proposed changes MUST be raised as a PR that edits this
  file, includes the rationale, updates the Sync Impact Report comment at the top,
  bumps the version, and updates `LAST_AMENDED_DATE`. At least one maintainer review
  is required to merge.
- **Versioning policy**: semantic versioning of this document.
  - **MAJOR**: a principle is removed, redefined incompatibly, or its NON-NEGOTIABLE
    status changes.
  - **MINOR**: a new principle or a new mandatory section is added, or existing
    guidance is materially expanded.
  - **PATCH**: clarifications, wording, typo fixes, non-semantic refinements.
- **Compliance review**: every `/speckit-plan` run MUST evaluate the planned design
  against the principles above in its Constitution Check gate; violations MUST be
  listed in Complexity Tracking with an explicit justification or the design MUST be
  revised. Periodic audits of merged code against these principles SHOULD be performed
  at least once per release cycle.
- For runtime development guidance and per-feature context, refer to `CLAUDE.md` and
  the active feature's `plan.md`.

**Version**: 1.0.0 | **Ratified**: 2026-05-05 | **Last Amended**: 2026-05-05
