# Implementation Plan: Product Catalog

**Branch**: `001-product-catalog` | **Date**: 2026-09-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-product-catalog/spec.md`

**Note**: This plan produces design artifacts only (spec store scope). No application code is
written by this command or by this repo. Implementation will happen later in the sibling repo
`../luv2shop-product-service` (relative to this spec store's parent folder), guided by this plan,
`data-model.md`, and `contracts/openapi.yaml`.

## Summary

Shoppers need to browse, filter, search, and view details of products in the LUV2SHOP catalog,
owned by `product-service`. This plan covers a read-only REST API — paged product list, paged
category list and paged name search (mutually exclusive with category filter), a category list,
and single-product detail lookup with a "not found" response — served from data loaded once into
memory at startup from a bundled JSON file, per the current constitution's no-database stack
constraint (v1.1.0). Technical approach: a single Spring Boot 3 (Java 21) application with a
simple `controller -> service -> in-memory repository` layering, no persistence framework, and an
OpenAPI 3 contract authored in this repo as the source of truth for the REST surface.

## Technical Context

**Language/Version**: Java 21

**Primary Dependencies**: Spring Boot 3 (Spring Web for REST controllers; Spring Boot's built-in
Jackson for JSON (de)serialization, including loading the seed data file at startup)

**Storage**: No database. Products and categories are loaded from
`src/main/resources/data/products.json` (bundled in the `product-service` repo) into an in-memory
store at application startup; all reads are served from memory.

**Testing**: JUnit 5 with Spring Boot Test (`@SpringBootTest` / `@WebMvcTest` + MockMvc) in the
`product-service` repo, per Constitution Principle IV (one test per acceptance criterion)

**Target Platform**: JVM server process (Spring Boot embedded Tomcat), deployable as a standalone
service; no OS-specific behavior

**Project Type**: Web service (single backend service, no frontend in this feature)

**Performance Goals**: List/category/search/detail responses under 2 seconds under normal load
(spec SC-001); supports at least 10,000 products without degrading response times (spec SC-004) —
achievable in-memory with simple linear/indexed lookups at this data scale

**Constraints**: No database engine, install, or login (constitution Technology Stack
Constraints v1.1.0); page size bounded to a maximum of 50 (spec, Clarifications); default page
size 20; default sort by product name ascending; name search and category filter are mutually
exclusive in the same request (category is ignored when a name search term is present)

**Scale/Scope**: Seed dataset of ~20 sample products across 4 categories, prices in INR; designed
to scale in-memory to at least 10,000 products (spec SC-004) without a redesign

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle / Constraint | Check | Result |
|---|---|---|
| I. Spec-First Development | `spec.md` for this feature exists and is approved before this plan/any code | PASS |
| II. Contract-Driven Service Communication | The `product-service` REST surface is defined as an OpenAPI 3 contract in this repo (`contracts/openapi.yaml`) before implementation | PASS (produced in Phase 1) |
| III. Service & Data Independence | `product-service` owns `products.json` exclusively in its own repo; no other service reads it; no shared storage | PASS |
| IV. Test-Backed Acceptance Criteria | Every FR/acceptance scenario in spec.md must map to an automated test in `product-service`; this plan defers actual test authoring to `/speckit-tasks` + implementation, but `quickstart.md` enumerates the scenarios to cover | PASS (planned, not yet implemented — expected at this stage) |
| V. Simplicity (YAGNI) | `controller -> service -> in-memory repository`, no DB, no speculative abstractions (no caching layer, no pagination library beyond simple manual paging) | PASS |
| Technology Stack Constraints (Java 21 / Spring Boot 3 / no database, local JSON, in-memory) | Matches Technical Context above exactly | PASS |

No violations identified. Complexity Tracking table is not needed.

## Project Structure

### Documentation (this feature)

```text
specs/001-product-catalog/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md         # Phase 1 output (/speckit-plan command)
├── quickstart.md         # Phase 1 output (/speckit-plan command)
├── contracts/
│   └── openapi.yaml      # Phase 1 output (/speckit-plan command)
└── tasks.md              # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (target repo — NOT created by this command)

This spec store repo does not contain `product-service` source code. The implementation lives in
the sibling repository `../luv2shop-product-service` (relative to this spec store's parent
folder), to be created/updated later by `/speckit-tasks` + implementation work in that repo. The
expected layout there, for reference by future tasks:

```text
luv2shop-product-service/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/.../productservice/
│   │   │   ├── controller/     # REST controllers (ProductController, CategoryController)
│   │   │   ├── service/        # ProductService, CategoryService (business/query logic)
│   │   │   ├── repository/     # In-memory repository loaded from JSON at startup
│   │   │   ├── model/          # Product, Category domain/DTO types
│   │   │   └── ProductServiceApplication.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── data/
│   │           └── products.json   # ~20 sample products, 4 categories, prices in INR
│   └── test/
│       └── java/.../productservice/
│           ├── controller/     # MockMvc contract/acceptance tests per FR
│           └── service/        # Unit tests for filtering/paging/sorting/search rules
```

**Structure Decision**: Single Spring Boot web service project (`luv2shop-product-service`), no
frontend/mobile component in this feature. Within it, a simple three-layer structure —
`controller -> service -> in-memory repository` — as required by Constitution Principle V
(Simplicity) and the "keep it simple" guidance for this feature. This spec store repo only holds
the plan, data model, OpenAPI contract, and quickstart for that implementation.

## Complexity Tracking

*No entries — Constitution Check reported no violations.*
