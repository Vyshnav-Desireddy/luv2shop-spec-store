<!--
Sync Impact Report
==================
Version change: 1.0.0 → 1.1.0
Bump rationale: MINOR — Technology Stack Constraints materially changed (MySQL replaced with local
  JSON file storage loaded into memory at startup); Principle III reworded to be storage-agnostic
  so it does not need to change again on future storage swaps. No principle was removed or
  redefined in a backward-incompatible way; Principle III's independence guarantee is unchanged.
Modified principles:
  - III. Service & Data Independence → wording generalized from "database"/"database schema" to
    "data store" so it covers the current local-JSON-file storage without implying a database
    exists; the independence rule itself (no cross-service reads/writes, no shared storage) is
    unchanged.
Added sections: none
Removed sections: none
Modified sections:
  - Technology Stack Constraints: removed MySQL/database requirement; added local JSON file
    storage per service, loaded into memory at startup, no database install or login required.
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
own their own data store, whatever concrete form it currently takes (see Technology Stack
Constraints). No service may read from or write to another service's data store directly, and no
storage — schema, files, or otherwise — may be shared across services. All cross-service data
access MUST go through the REST APIs defined under Principle II.
Rationale: Independent ownership of code and data is what makes these three deployable units
actual microservices rather than a distributed monolith; direct storage coupling defeats that
boundary silently and is hard to detect later, regardless of what storage technology is in use.

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
- **Data Storage**: No database, for now. Each service stores its data in local JSON files inside
  its own repo, loaded into memory at startup. There is no database engine to install and no
  database login/credentials to manage. Each service still owns its own data files exclusively and
  MUST NOT read another service's data files (see Principle III); this is a storage mechanism
  change only, not a relaxation of data independence.

Specs and OpenAPI contracts authored in this repo MUST be technically feasible within this stack.
Any proposal to deviate from this stack for a given service — including reintroducing a database —
requires an explicit constitution amendment (see Governance) before it can be reflected in that
service's specs.

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

**Version**: 1.1.0 | **Ratified**: 2026-09-28 | **Last Amended**: 2026-09-28
