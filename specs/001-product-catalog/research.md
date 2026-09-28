# Phase 0 Research: Product Catalog

All technical inputs for this feature were supplied directly (Java 21, Spring Boot 3, Maven,
Spring Web, no database, JSON seed file, REST under `/api`, page/size paging). No
`NEEDS CLARIFICATION` markers remain in the Technical Context. The items below record the concrete
decisions made to turn those inputs into a coherent, simple design, per Constitution Principle V
(Simplicity/YAGNI).

## 1. In-memory data loading

- **Decision**: On startup, read `src/main/resources/data/products.json` once (e.g., via a
  `CommandLineRunner`/`ApplicationRunner` bean or a constructor-time load in the repository bean),
  deserialize it into plain in-memory Java objects (`List<Product>`, `List<Category>`), and hold
  them in a simple in-memory repository component for the lifetime of the process.
- **Rationale**: Matches the constitution's no-database constraint exactly; a single load at
  startup is the simplest mechanism that satisfies "loaded into memory at startup" with no
  reload/watch complexity, which isn't required by the spec.
- **Alternatives considered**: Lazy/on-demand loading per request (rejected — adds complexity and
  repeated I/O for no benefit at this data scale); an embedded database like H2 (rejected — the
  constitution explicitly removed the database requirement for now).

## 2. In-memory repository shape

- **Decision**: Keep two flat, pre-built in-memory structures: a `List<Product>` sorted by name
  ascending (the default and only sort order per spec FR-010), and a `List<Category>`. Category
  filtering and name search are simple linear scans/filters over the sorted product list, which
  preserves name ordering without re-sorting per request.
- **Rationale**: At the stated scale (seed of ~20 products, designed to hold up to 10,000 per spec
  SC-004), a linear scan comfortably meets the <2s response goal (SC-001) without needing indexes,
  a search library, or a database. Building the list pre-sorted avoids repeated sort work per
  request.
- **Alternatives considered**: Per-category or per-id `Map` indexes (rejected for now as
  unnecessary complexity beyond a lookup-by-id map, which is the one index actually needed);
  a full-text search library (rejected — out of proportion to "search by name" substring matching).
  A `Map<String, Product>` keyed by id is retained for O(1) product-detail lookups (FR-005/FR-006).

## 3. Paging

- **Decision**: Manual paging via `page` (0-based) and `size` query parameters on `/api/products`
  and `/api/products/{category-or-search variant}`. Default `size` = 20, maximum allowed `size` =
  50 (spec Clarifications); requests beyond the last page return an empty result set, not an
  error (spec FR-008). Response includes the page content plus paging metadata (page, size, total
  items, total pages).
- **Rationale**: Directly matches the spec's explicit paging clarifications; a hand-built paging
  DTO is simpler than pulling in Spring Data's `Pageable`/`Page<T>` machinery (which assumes a
  `Repository`/datastore abstraction this feature does not use), consistent with Simplicity.
- **Alternatives considered**: Spring Data `Pageable` (rejected — designed for Spring Data
  repositories backed by a real datastore; would be an unused abstraction layer here); cursor-based
  paging (rejected — no requirement for it, adds complexity).

## 4. Category filter vs. name search exclusivity

- **Decision**: `/api/products` accepts optional `category` and `name` query parameters. If `name`
  is present, the search runs across the whole catalog and `category` (if also supplied) is
  ignored — never combined — per spec FR-004 and User Story 5 acceptance scenario 3.
- **Rationale**: This directly implements the clarified requirement without introducing a 400/error
  response the spec never asked for; ignoring the redundant parameter is simpler than adding
  request validation for a combination that is merely irrelevant, not invalid input.
- **Alternatives considered**: Reject requests supplying both parameters with `400 Bad Request`
  (rejected — spec frames this as "not combined," not as a client error to reject).

## 5. Not-found and empty-result handling

- **Decision**: `GET /api/products/{id}` returns `404 Not Found` with a small JSON error body
  (e.g., `{"message": "Product not found"}`) when the id doesn't exist (FR-006/SC-002). Category
  filters, name searches, or the categories list that match nothing return `200 OK` with an empty
  paged/list result, never an error (FR-007/SC-003).
- **Rationale**: Matches standard REST conventions and the spec's explicit distinction between "no
  such single resource" (404) and "an empty collection of matches" (200 + empty list).
- **Alternatives considered**: Returning `200` with a null body for a missing product (rejected —
  spec explicitly requires shoppers be "told" not found, and 404 is the conventional, unambiguous
  way to say that in a REST API).

## 6. Sorting

- **Decision**: All list-producing endpoints (full list, category list, name search) sort results
  by product name ascending by default, with no other sort option exposed in this feature
  (FR-010; no spec requirement for alternate sorts).
- **Rationale**: Matches the clarified requirement exactly; adding sort-parameter flexibility now
  would be speculative given YAGNI.
- **Alternatives considered**: Exposing a generic `sort` query parameter (rejected — not required
  by the spec; would be unused complexity for now).

## 7. Testing approach

- **Decision**: Use Spring Boot Test with MockMvc (`@WebMvcTest` or a full `@SpringBootTest` with
  `MockMvc`) for controller-level acceptance tests mapped 1:1 to spec acceptance scenarios, plus
  plain JUnit 5 unit tests for the repository/service filtering, paging, sorting, and exclusivity
  rules. This satisfies Constitution Principle IV (every acceptance criterion has a test).
- **Rationale**: Standard, minimal-dependency testing approach for a Spring Boot Web project;
  no additional test framework is needed.
- **Alternatives considered**: Full integration tests spinning up a real HTTP server with
  `TestRestTemplate`/`WebTestClient` (viable but heavier than needed given there is no database or
  external system to integrate with; MockMvc is sufficient and simpler).

## 8. OpenAPI contract authoring

- **Decision**: Hand-author `contracts/openapi.yaml` as a plain OpenAPI 3.0 YAML document
  describing `/api/products`, `/api/products/{id}`, and `/api/categories`, matching the
  request/response shapes decided above. No code-generation tooling is introduced in this plan.
- **Rationale**: Constitution Principle II requires the contract to exist in this repo as the
  source of truth; hand-authoring keeps the plan phase free of implementation tooling decisions,
  which belong to the `product-service` repo's own build if it later chooses codegen.
- **Alternatives considered**: Generating the OpenAPI file from annotated controller code
  (rejected — that code doesn't exist yet, and would invert the contract-first principle).

**Output**: All Technical Context items are resolved; no `NEEDS CLARIFICATION` markers remain.
