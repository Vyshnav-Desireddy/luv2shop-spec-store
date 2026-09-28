<!--
Sync Impact Report
==================
Version change: (none) → 1.0.0
Bump rationale: Initial ratification — first formal adoption of the constitution for this project.
Modified principles: N/A (initial creation)
Added sections:
  - Core Principles: I. Spec-First Development, II. Contract-Driven Service Communication,
    III. Service & Data Independence, IV. Test-Backed Acceptance Criteria, V. Simplicity (YAGNI)
  - Technology Stack Constraints
  - Development Workflow
  - Governance
Removed sections: N/A (initial creation)
Follow-up TODOs: none
==================
-->

# LUV2SHOP Constitution

## Core Principles

### I. Spec-First Development (NON-NEGOTIABLE)
No code is written for any LUV2SHOP service until a corresponding spec exists in this repository
(`luv2shop-spec-store`). This repo is the single source of truth for what the system does and how
its services behave. Every feature, endpoint, or behavior change MUST originate from a spec here
before implementation begins in `product-service`, `order-service`, `customer-service`, or any
other service repo.
Rationale: Splitting implementation across independent service repos only stays coherent if there
is one authoritative place defining intent before code exists; otherwise repos drift and
"tribal knowledge" replaces documented behavior.

### II. Contract-Driven Service Communication
Services communicate with each other exclusively through REST APIs, and every such API MUST be
defined as an OpenAPI contract stored in this repo. No service may call another service through a
shared library, direct database access, message format, or any channel not described by an OpenAPI
contract here. Contract changes MUST be made in this repo first, then implemented in the owning
service.
Rationale: With three independently deployable services, the OpenAPI contracts in the spec store
are the only reliable, versionable record of the integration surface between them.

### III. Service & Data Independence
`product-service`, `order-service`, and `customer-service` each live in their own repository and
own their own database. No service may read from or write to another service's database directly,
and no database schema may be shared across services. All cross-service data access MUST go
through the REST APIs defined under Principle II.
Rationale: Independent ownership of code and data is what makes these three deployable units
actual microservices rather than a distributed monolith; direct database coupling defeats that
boundary silently and is hard to detect later.

### IV. Test-Backed Acceptance Criteria (NON-NEGOTIABLE)
Every acceptance criterion defined in a spec MUST have at least one corresponding automated test
in the implementing service repo. A feature is not considered complete, and a spec is not
considered satisfied, until its acceptance criteria are each demonstrably covered by a test.
Rationale: Acceptance criteria without tests are just prose; requiring a test per criterion is
what makes "done" verifiable rather than asserted.

### V. Simplicity (YAGNI)
Specs, contracts, and the implementations that follow them MUST favor the simplest design that
satisfies the current acceptance criteria. Do not introduce speculative abstractions, extra
services, extra endpoints, or extra configuration for hypothetical future needs. Complexity that
is not justified by a current, documented requirement MUST be removed or avoided.
Rationale: A 3-service ecommerce system stays maintainable only if additions are driven by actual
requirements in the spec store, not anticipated ones.

## Technology Stack Constraints

All three services (`product-service`, `order-service`, `customer-service`) MUST be built on:
- **Language/Runtime**: Java 21
- **Framework**: Spring Boot 3
- **Database**: MySQL, one dedicated schema/instance per service (see Principle III)

Specs and OpenAPI contracts authored in this repo MUST be technically feasible within this stack.
Any proposal to deviate from this stack for a given service requires an explicit constitution
amendment (see Governance) before it can be reflected in that service's specs.

## Development Workflow

1. A change starts as a spec in this repo (`luv2shop-spec-store`), including its acceptance
   criteria, before any implementation work is planned.
2. If the change involves cross-service communication, the relevant OpenAPI contract(s) in this
   repo MUST be authored or updated as part of the same spec change.
3. Implementation happens in the owning service's own repository, guided by the spec and the
   OpenAPI contract, using the stack defined above.
4. Each acceptance criterion in the spec MUST map to at least one automated test before the
   implementing change is considered complete.
5. Reviews of specs, contracts, and implementing code MUST verify compliance with all five Core
   Principles above; any exception MUST be documented and justified in the spec itself.

## Governance

This constitution supersedes any conflicting practice, template, or informal convention used
across the LUV2SHOP repos. Where a spec, plan, or task template conflicts with this document, this
document wins.

**Amendment procedure**: Amendments are proposed by editing this file, describing the change and
its rationale, and updating the Sync Impact Report at the top of the file. Amendments take effect
once merged into this repo.

**Versioning policy**: This constitution is versioned independently using semantic versioning:
- **MAJOR**: Backward-incompatible governance changes, or removal/redefinition of a principle.
- **MINOR**: A new principle or section added, or materially expanded guidance.
- **PATCH**: Clarifications, wording fixes, or non-semantic refinements.

**Compliance review**: Every spec, plan, and set of tasks produced in this repo, and every PR in a
service repo implementing them, MUST be checked against the Core Principles above. Any deviation
MUST be called out explicitly and justified rather than silently introduced.

**Version**: 1.0.0 | **Ratified**: 2026-09-28 | **Last Amended**: 2026-09-28
