# Quickstart: Validating the Product Catalog Feature

This guide describes how to run and manually validate `product-service` once it is implemented in
the sibling repo `../luv2shop-product-service`, against this feature's spec and contract. It does
not contain implementation code — see `data-model.md` for the domain shapes and
`contracts/openapi.yaml` for the authoritative API contract.

## Prerequisites

- Java 21 JDK installed.
- Maven installed (or use the repo's Maven wrapper once created).
- `luv2shop-product-service` repo checked out as a sibling of this spec store
  (`../luv2shop-product-service`), with `src/main/resources/data/products.json` populated with the
  seed dataset (~20 products across 4 categories, prices in INR — see `data-model.md`).

## Run the service

From within `luv2shop-product-service`:

```bash
mvn spring-boot:run
```

The service starts on its configured port (e.g., `http://localhost:8080`) and loads
`products.json` into memory at startup — no database setup or login required.

## Validation scenarios

Each scenario below maps to an acceptance scenario in `spec.md`. Replace `<id>` /
`<category-id>` / `<term>` with real values from the seed data.

### 1. Browse the full product list, paged, sorted by name (User Story 1)

```bash
curl "http://localhost:8080/api/products?page=0&size=20"
```

- **Expect**: `200 OK`, `items` sorted by `name` ascending, `totalItems`/`totalPages` reflecting
  the full seed dataset size.

```bash
curl "http://localhost:8080/api/products?page=1&size=20"
```

- **Expect**: the next page of products (or an empty `items` array if there is no next page —
  never an error).

### 2. View a single product's details (User Story 2)

```bash
curl "http://localhost:8080/api/products/<id>"
```

- **Expect**: `200 OK` with `name`, `description`, `price` (INR), `imageUrl`, `unitsInStock`.

```bash
curl "http://localhost:8080/api/products/does-not-exist"
```

- **Expect**: `404 Not Found` with an `ErrorResponse` body, e.g. `{"message": "Product not found"}`.

### 3. Filter by category (User Story 3)

```bash
curl "http://localhost:8080/api/products?category=<category-id>&page=0&size=20"
```

- **Expect**: `200 OK`, only products in that category, sorted by name ascending.

```bash
curl "http://localhost:8080/api/products?category=empty-or-unknown-category"
```

- **Expect**: `200 OK` with an empty `items` array (not an error), whether the category exists
  with no products or doesn't exist at all.

### 4. List all categories (User Story 4)

```bash
curl "http://localhost:8080/api/categories"
```

- **Expect**: `200 OK` with all 4 seed categories.

### 5. Search products by name (User Story 5)

```bash
curl "http://localhost:8080/api/products?name=<term>&page=0&size=20"
```

- **Expect**: `200 OK`, only products whose name contains `<term>` (case-insensitive), sorted by
  name ascending.

```bash
curl "http://localhost:8080/api/products?name=no-such-term"
```

- **Expect**: `200 OK` with an empty `items` array (not an error).

```bash
curl "http://localhost:8080/api/products?name=<term>&category=<category-id>"
```

- **Expect**: `200 OK`, results are the same as searching by `<term>` alone — `category` is
  ignored when `name` is present (FR-004).

## Contract validation

Validate the running service's responses against `contracts/openapi.yaml` using any OpenAPI
diff/validation tool of choice (e.g., point a schema-validating HTTP client or CI step at the
contract file); this is left to the `product-service` repo's own tooling choices.

## Automated tests

Per Constitution Principle IV, each acceptance scenario above must also exist as an automated test
in `luv2shop-product-service` (MockMvc controller tests + service/repository unit tests — see
`research.md` #7). These are authored during implementation (`/speckit-tasks` + build), not as part
of this quickstart.
