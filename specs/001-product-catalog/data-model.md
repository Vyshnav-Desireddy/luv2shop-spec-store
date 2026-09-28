# Data Model: Product Catalog

Derived from `spec.md` Key Entities and Functional Requirements. This describes the in-memory
domain shape for `product-service`; it is not a database schema (see constitution Technology Stack
Constraints v1.1.0 — no database, local JSON file loaded into memory at startup).

## Category

Represents a single grouping used to organize products and to populate the shopper-facing category
menu (spec User Story 4, FR-009).

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Unique, stable identifier; referenced by `Product.categoryId`. Used as the `category` query parameter value on `/api/products`. |
| `name` | string | yes | Display name of the category, e.g., "Dog Food". |

**Rules**:
- `id` is unique across all categories in `products.json`.
- The seed data must define exactly 4 categories (per plan Scope).
- The categories list is returned in full, unpaged (spec Assumptions: "small, complete list").

## Product

Represents a single catalog item (spec Key Entities, FR-001–FR-010).

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Unique, stable identifier; used for `/api/products/{id}` lookups. |
| `name` | string | yes | Product name; used for name search (FR-004) and default sort key (FR-010). |
| `description` | string | yes | Full text description, returned in product details (FR-005). |
| `price` | decimal | yes | Price in INR (spec Clarifications); non-negative. |
| `imageUrl` | string | yes | Reference to a displayable image (spec Assumptions: reference, not an upload). |
| `unitsInStock` | integer | yes | Non-negative; `0` is valid and must still be shown (spec Edge Cases). |
| `categoryId` | string | yes | Foreign key to `Category.id`; each product belongs to exactly one category (spec Assumptions). |

**Validation rules** (enforced when loading `products.json` at startup):
- `id` and `name` are non-empty and unique per product (`id` unique catalog-wide).
- `price >= 0`; `unitsInStock >= 0`.
- `categoryId` must reference an existing `Category.id` in the same file.

**Derived/query behavior** (not stored fields, computed by the service layer):
- Default list ordering: `name` ascending, case-insensitive (FR-010).
- Name search: case-insensitive substring match against `name` (research.md #2; spec Assumptions).
- Category filter and name search are mutually exclusive per request: if a name search term is
  present, any supplied category filter is ignored (FR-004; research.md #4).

## Relationships

```text
Category (1) ──< (many) Product
```

- Each `Product.categoryId` references exactly one `Category.id`.
- No product may exist without a valid category; no category is required to have products (an
  empty category is valid — spec Edge Cases).

## Seed Data Shape (`src/main/resources/data/products.json`)

Illustrative shape only (implementation detail for the `product-service` repo, not created here):

```json
{
  "categories": [
    { "id": "cat-books", "name": "Books" },
    { "id": "cat-electronics", "name": "Electronics" },
    { "id": "cat-clothing", "name": "Clothing" },
    { "id": "cat-home-kitchen", "name": "Home & Kitchen" }
  ],
  "products": [
    {
      "id": "prod-0001",
      "name": "Wireless Bluetooth Headphones",
      "description": "Over-ear wireless headphones with 30-hour battery life and active noise cancellation.",
      "price": 1999.00,
      "imageUrl": "https://example.com/images/prod-0001.jpg",
      "unitsInStock": 42,
      "categoryId": "cat-electronics"
    }
  ]
}
```

Seed data must contain ~20 products spread across the 4 categories above, with prices in INR
(plan Scope/Scale).

## Response DTO Shapes (service/API level, not persisted)

These are the shapes the service layer produces for the controller layer to return; see
`contracts/openapi.yaml` for the authoritative schema.

- **ProductSummary** (used in list/search/category results): `id`, `name`, `price`, `imageUrl`,
  `unitsInStock`, `categoryId`.
- **ProductDetail** (used in single-product lookup): all `ProductSummary` fields plus
  `description`.
- **Page<T>**: `items` (array of `T`), `page`, `size`, `totalItems`, `totalPages`.
- **CategoryList**: plain array of `Category`.
- **ErrorResponse**: `message` (string) — used for the 404 "product not found" case (FR-006).
