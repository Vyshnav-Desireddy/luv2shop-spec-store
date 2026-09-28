---

description: "Task list template for feature implementation"
---

# Tasks: Product Catalog

**Input**: Design documents from `/specs/001-product-catalog/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml, quickstart.md

**Tests**: Included and REQUIRED — Constitution Principle IV (Test-Backed Acceptance Criteria,
NON-NEGOTIABLE) mandates at least one automated test per acceptance criterion; this is not the
optional default.

**Organization**: Tasks are grouped by user story (from spec.md) to enable independent
implementation and testing of each story.

**Target repo**: All implementation file paths below are in the sibling repository
`../luv2shop-product-service` (relative to this spec store), per plan.md — this spec store repo
contains no application code. Package: `com.luv2shop.productservice` (reasonable default; adjust
if the target repo's actual group/artifact id differs).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project (Spring Boot web service)**:
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/...` and
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/...`, per plan.md
  Project Structure.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure for `luv2shop-product-service`

- [ ] T001 Create Maven project skeleton in `../luv2shop-product-service/pom.xml`: Java 21 source/target,
  `spring-boot-starter-parent` 3.x, `spring-boot-starter-web` and `spring-boot-starter-test`
  dependencies (plan.md Technical Context: Java 21, Spring Boot 3, Maven, Spring Web)
- [ ] T002 [P] Create Spring Boot entry point in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/ProductServiceApplication.java`
- [ ] T003 [P] Add `../luv2shop-product-service/src/main/resources/application.yml` with app name and
  server port configuration

**Checkpoint**: Project builds and runs an empty Spring Boot app.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared data model, in-memory repository, and cross-cutting error handling that every
user story depends on. No user story work should begin until this phase is complete.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 [P] Create `Category` model in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/model/Category.java` with
  fields `id` (String, required, unique) and `name` (String, required) — data-model.md Category
- [ ] T005 [P] Create `Product` model in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/model/Product.java` with
  fields `id` (String, required, unique), `name` (String, required), `description` (String,
  required), `price` (BigDecimal, required, "price >= 0"), `imageUrl` (String, required),
  `unitsInStock` (int, required, "unitsInStock >= 0"), `categoryId` (String, required, "must
  reference an existing Category.id in the same file") — data-model.md Product
- [ ] T006 Create seed data file
  `../luv2shop-product-service/src/main/resources/data/products.json` with exactly 4 categories
  (Books, Electronics, Clothing, Home & Kitchen) and ~20 products spread across them, all prices in
  INR, matching the `Category`/`Product` shapes from T004/T005 (plan.md Scope/Scale; data-model.md
  Seed Data Shape)
- [ ] T007 Implement `CatalogRepository` in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/repository/CatalogRepository.java`:
  on construction/startup, load and deserialize `data/products.json` once via Jackson into an
  in-memory `List<Product>` pre-sorted by `name` ascending (case-insensitive), a `List<Category>`,
  and a `Map<String, Product>` keyed by `id` for O(1) detail lookups; validate on load that every
  `Product.categoryId` references an existing `Category.id` (research.md #1, #2; data-model.md
  Validation rules) (depends on T004, T005, T006)
- [ ] T008 [P] Create response DTOs in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/model/dto/`:
  `ProductSummary` (`id`, `name`, `price`, `imageUrl`, `unitsInStock`, `categoryId`),
  `ProductDetail` (all `ProductSummary` fields plus `description`), `ProductSummaryPage` (`items`,
  `page`, `size`, `totalItems`, `totalPages`), `ErrorResponse` (`message`) — data-model.md Response
  DTO Shapes; contracts/openapi.yaml schemas
- [ ] T009 Implement a `@ControllerAdvice` exception handler in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/controller/ApiExceptionHandler.java`
  mapping a "product not found" condition to `404` with an `ErrorResponse` body (FR-006;
  contracts/openapi.yaml `/products/{id}` 404 response), and invalid paging parameters to `400`
  with an `ErrorResponse` body (contracts/openapi.yaml `/products` 400 response)

**Checkpoint**: Foundation ready — in-memory data is loaded and validated at startup; user story
implementation can now begin.

---

## Phase 3: User Story 1 - Browse Product Catalog (Priority: P1) 🎯 MVP

**Goal**: Shoppers can retrieve a paged list of all products, sorted by name ascending by default.

**Independent Test**: Request `/api/products` with and without `page`/`size` and confirm a bounded,
correctly sorted page of products is returned, with subsequent pages reachable and an empty (not
error) result past the last page.

### Tests for User Story 1 (REQUIRED — Constitution Principle IV)

> Write these tests FIRST, ensure they FAIL before implementation

- [ ] T010 [P] [US1] Contract test in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/controller/ProductListControllerTest.java`
  covering spec.md US1 acceptance scenarios 1–4: first page with more-pages indication, next page
  navigation, single page when catalog fits, default name-ascending sort (FR-001, FR-002, FR-010)
- [ ] T011 [P] [US1] Unit test in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/service/ProductPagingTest.java`
  covering default page size 20, maximum page size 50, and an empty page when requesting beyond the
  last page (FR-001, FR-008; spec.md Clarifications max page size = 50)

### Implementation for User Story 1

- [ ] T012 [US1] Implement `ProductService.listProducts(int page, int size)` in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/service/ProductService.java`:
  returns a name-ascending-sorted, paged `ProductSummaryPage`, defaulting `size` to 20, clamping/
  rejecting `size` above 50, and returning an empty page when `page` is beyond the last page
  (depends on T007, T008)
- [ ] T013 [US1] Implement `ProductController` with `GET /api/products` (no filters yet) in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/controller/ProductController.java`,
  binding `page` (default 0) and `size` (default 20, max 50) query params to `ProductService`
  (depends on T012)
- [ ] T014 [US1] Add `page`/`size` request validation (page ≥ 0; 1 ≤ size ≤ 50) in
  `ProductController`, returning `400` with `ErrorResponse` via T009's handler on violation
  (depends on T013, T009)

**Checkpoint**: At this point, User Story 1 (browse, paged, sorted) should be fully functional and
testable independently — this is the MVP.

---

## Phase 4: User Story 2 - View Product Details (Priority: P1)

**Goal**: Shoppers can open a single product to see its full details, or be told it was not found.

**Independent Test**: Request `/api/products/{id}` for a known id and confirm all detail fields are
returned; request an unknown id and confirm a `404` "not found" response.

### Tests for User Story 2 (REQUIRED — Constitution Principle IV)

- [ ] T015 [P] [US2] Contract test in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/controller/ProductDetailControllerTest.java`
  covering spec.md US2 acceptance scenarios: `200` with name/description/price/imageUrl/
  unitsInStock for an existing product id, and `404` with an `ErrorResponse` for a non-existent id
  (FR-005, FR-006; SC-002)

### Implementation for User Story 2

- [ ] T016 [US2] Implement `ProductService.getById(String id)` returning an `Optional<ProductDetail>`
  (empty when not found) in `ProductService.java` (depends on T007, T008)
- [ ] T017 [US2] Add `GET /api/products/{id}` to `ProductController`, returning `200` with
  `ProductDetail` when present or delegating to the `404` handler from T009 when absent (depends on
  T016, T009, T013)

**Checkpoint**: User Stories 1 AND 2 (the two P1 flows) both work independently.

---

## Phase 5: User Story 3 - Filter by Category (Priority: P2)

**Goal**: Shoppers can retrieve a paged list of products belonging to a single category.

**Independent Test**: Request `/api/products?category=<id>` for a category with products and
confirm only that category's products are returned, sorted by name; request a category with no
products (or an unknown category id) and confirm an empty result, not an error.

### Tests for User Story 3 (REQUIRED — Constitution Principle IV)

- [ ] T018 [P] [US3] Contract test in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/controller/ProductCategoryFilterControllerTest.java`
  covering spec.md US3 acceptance scenarios: only-matching-category results sorted by name, and an
  empty result for a category with no products and for a category id that doesn't exist at all
  (FR-003, FR-007; Edge Cases)

### Implementation for User Story 3

- [ ] T019 [US3] Extend `ProductService.listProducts` to accept an optional `categoryId` filter,
  still sorted and paged, in `ProductService.java` (depends on T012)
- [ ] T020 [US3] Extend `GET /api/products` in `ProductController` to accept an optional `category`
  query param (depends on T019, T013)

**Checkpoint**: User Stories 1, 2, and 3 are all independently functional.

---

## Phase 6: User Story 4 - View All Categories (Priority: P2)

**Goal**: Shoppers (via the storefront) can retrieve the full list of categories for a category
menu.

**Independent Test**: Request `/api/categories` and confirm all categories in the catalog are
returned; confirm an empty array (not an error) if no categories exist.

### Tests for User Story 4 (REQUIRED — Constitution Principle IV)

- [ ] T021 [P] [US4] Contract test in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/controller/CategoryControllerTest.java`
  covering spec.md US4 acceptance scenarios: full category list returned, and empty array when no
  categories exist (FR-009)

### Implementation for User Story 4

- [ ] T022 [US4] Implement `CategoryService.listAll()` returning all categories from
  `CatalogRepository` in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/service/CategoryService.java`
  (depends on T007)
- [ ] T023 [US4] Implement `CategoryController` with `GET /api/categories` in
  `../luv2shop-product-service/src/main/java/com/luv2shop/productservice/controller/CategoryController.java`
  (depends on T022)

**Checkpoint**: User Stories 1–4 are all independently functional.

---

## Phase 7: User Story 5 - Search Products by Name (Priority: P2)

**Goal**: Shoppers can search for products by name across the whole catalog, independent of any
category filter.

**Independent Test**: Request `/api/products?name=<term>` and confirm matching products (case-
insensitive substring) are returned, paged and sorted by name; confirm an empty result for a
non-matching term; confirm that supplying both `name` and `category` ignores `category`.

### Tests for User Story 5 (REQUIRED — Constitution Principle IV)

- [ ] T024 [P] [US5] Contract test in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/controller/ProductSearchControllerTest.java`
  covering spec.md US5 acceptance scenarios: matching-term results sorted by name, empty result for
  a non-matching term, and identical results whether or not a `category` param is also supplied
  (FR-004; US5 acceptance scenario 3)
- [ ] T025 [P] [US5] Unit test in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/service/ProductSearchTest.java`
  covering case-insensitive substring matching and that `categoryId` is ignored whenever a `name`
  term is present (FR-004; research.md #4)

### Implementation for User Story 5

- [ ] T026 [US5] Extend `ProductService.listProducts` to accept an optional `name` search term,
  performing a case-insensitive substring match across the whole catalog and taking precedence over
  any supplied `categoryId` (i.e., `categoryId` is ignored when `name` is present), in
  `ProductService.java` (depends on T019)
- [ ] T027 [US5] Extend `GET /api/products` in `ProductController` to accept an optional `name`
  query param (depends on T026, T020)

**Checkpoint**: All five user stories are independently functional.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T028 [P] Add `../luv2shop-product-service/README.md` documenting how to build and run the
  service, referencing quickstart.md's validation scenarios
- [ ] T029 [P] Add model validation unit tests in
  `../luv2shop-product-service/src/test/java/com/luv2shop/productservice/model/ProductValidationTest.java`
  for the data-model.md Validation rules: non-negative `price`, non-negative `unitsInStock`, and
  `categoryId` referential integrity at load time
- [ ] T030 Manually run all quickstart.md validation scenarios against the running service and
  confirm each matches its expected result
- [ ] T031 Cross-check implemented request/response shapes against contracts/openapi.yaml
  (`/products`, `/products/{id}`, `/categories`) for schema drift

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **User Stories (Phase 3–7)**: All depend on Foundational phase completion
  - US1 and US2 (both P1) have no dependency on each other and can proceed in parallel
  - US3, US4, US5 (all P2) each build on the shared `ProductService.listProducts` introduced in
    US1 (T012); US3 extends it (T019), and US5 further extends US3's extension (T026) — so within
    `ProductService`/`ProductController`, US3 → US5 is a sequential file-level dependency, but each
    story remains independently testable once its own tasks land
  - US4 (categories) has no dependency on US1/US2/US3/US5 beyond the shared Foundational phase
- **Polish (Phase 8)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational — no dependency on other stories
- **User Story 2 (P1)**: Can start after Foundational — no dependency on other stories
- **User Story 3 (P2)**: Can start after Foundational; implementation tasks (T019–T020) build on
  US1's `ProductService`/`ProductController` (T012–T013), so schedule after US1 if working
  sequentially
- **User Story 4 (P2)**: Can start after Foundational — no dependency on other stories
- **User Story 5 (P2)**: Can start after Foundational; implementation tasks (T026–T027) build on
  US3's extensions (T019–T020), so schedule after US3 if working sequentially

### Within Each User Story

- Tests MUST be written and FAIL before implementation
- Models/DTOs (Foundational) before services
- Services before controllers/endpoints
- Core implementation before cross-story extension

### Parallel Opportunities

- T002, T003 (Setup) can run in parallel
- T004, T005, T008 (Foundational models/DTOs) can run in parallel; T006 and T007 are sequential
  after them
- Once Foundational (Phase 2) completes: US1 and US2 can be staffed in parallel; US4 can also run
  in parallel with either; US3 should follow US1's service/controller tasks, and US5 should follow
  US3
- Test tasks within a story marked [P] (e.g., T010/T011, T024/T025) can run in parallel with each
  other

---

## Parallel Example: User Story 1

```bash
# Launch both tests for User Story 1 together:
Task: "Contract test in .../controller/ProductListControllerTest.java (T010)"
Task: "Unit test in .../service/ProductPagingTest.java (T011)"
```

## Parallel Example: Foundational Phase

```bash
# Launch independent model/DTO tasks together:
Task: "Create Category model in .../model/Category.java (T004)"
Task: "Create Product model in .../model/Product.java (T005)"
Task: "Create response DTOs in .../model/dto/ (T008)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1 (browse, paged, sorted)
4. **STOP and VALIDATE**: Run quickstart.md scenario 1 against the running service
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → in-memory catalog loads and validates at startup
2. Add User Story 1 → test independently → MVP (browsing works)
3. Add User Story 2 → test independently → product detail pages work
4. Add User Story 3 → test independently → category filtering works
5. Add User Story 4 → test independently → category menu works
6. Add User Story 5 → test independently → name search works
7. Polish: docs, model validation tests, quickstart + contract cross-check

### Parallel Team Strategy

With multiple developers, after Setup + Foundational:
- Developer A: User Story 1, then User Story 3, then User Story 5 (they share
  `ProductService.listProducts`)
- Developer B: User Story 2 (independent)
- Developer C: User Story 4 (independent)
