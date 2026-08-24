# Product Inventory API

A RESTful inventory management service built with Spring Boot 3 and Java 21. It exposes full CRUD over a `Product` resource, enforces bean validation and SKU business rules at the service layer, and returns consistent HTTP status codes through a centralized exception handler.

> Ostad Batch 03 — Module 14 Assignment

---

## Tech Stack

| Layer | Choice |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.5.11 |
| Persistence | Spring Data JPA (Hibernate) |
| Database | H2 (in-memory) |
| Validation | Jakarta Bean Validation (`spring-boot-starter-validation`) |
| Boilerplate | Lombok |
| Build | Maven (wrapper included) |

---

## Getting Started

### Prerequisites

- JDK 21 or newer
- No Maven install required — use the bundled wrapper

### Run

```bash
git clone https://github.com/FatemaTujZohra20/Product-Inventory-API-Java_SpringBoot_Ostad_Batch03_Module14_Assignment.git
cd Product-Inventory-API-Java_SpringBoot_Ostad_Batch03_Module14_Assignment

# Linux / macOS
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

The API starts on **http://localhost:8080**.

### Build a JAR

```bash
./mvnw clean package
java -jar target/ProductInventoryAPI-0.0.1-SNAPSHOT.jar
```

### H2 Console

Available at **http://localhost:8080/h2-console**

| Field | Value |
|---|---|
| JDBC URL | `jdbc:h2:mem:productinventorydb` |
| Username | `sa` |
| Password | `password` |

The database is in-memory, so all data is discarded when the application stops.

---

## Data Model

### `Product`

| Field | Type | Constraints |
|---|---|---|
| `id` | `Long` | Auto-generated (identity) |
| `name` | `String` | Must not be blank |
| `description` | `String` | Max 500 characters |
| `sku` | `String` | Must not be blank, unique, format `SKU-XXXXXXXX` |
| `price` | `Double` | Required, must be positive |
| `quantity` | `Integer` | Required, minimum 0 |
| `status` | `ProductStatus` | Required, persisted as string |

### `ProductStatus`

`ACTIVE` · `INACTIVE` · `DISCONTINUED`

---

## API Reference

Base path: `/api/products`

| Method | Endpoint | Description | Success |
|---|---|---|---|
| `POST` | `/api/products` | Create a product | `201 Created` |
| `GET` | `/api/products` | List all products | `200 OK` |
| `GET` | `/api/products/{id}` | Fetch a product by ID | `200 OK` |
| `PUT` | `/api/products/{id}` | Update a product | `200 OK` |
| `DELETE` | `/api/products/{id}` | Delete a product | `204 No Content` |

### Create a product

```bash
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Wireless Gaming Mouse",
    "description": "Ergonomic 16000 DPI optical sensor mouse with RGB lighting.",
    "sku": "SKU-A1B2C3D4",
    "price": 59.99,
    "quantity": 25,
    "status": "ACTIVE"
  }'
```

**201 Created**

```json
{
  "id": 1,
  "name": "Wireless Gaming Mouse",
  "description": "Ergonomic 16000 DPI optical sensor mouse with RGB lighting.",
  "sku": "SKU-A1B2C3D4",
  "price": 59.99,
  "quantity": 25,
  "status": "ACTIVE"
}
```

### Update a product

```bash
curl -X PUT http://localhost:8080/api/products/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Wireless Gaming Mouse - V2",
    "description": "Updated version with longer battery life.",
    "sku": "SKU-A1B2C3D4",
    "price": 64.99,
    "quantity": 50,
    "status": "ACTIVE"
  }'
```

The `sku` in the request body must match the stored SKU — SKUs are immutable once assigned.

---

## Business Rules

Validation happens in two places: annotation-driven bean validation on the entity, and explicit rules in `ProductService`.

1. **SKU format** — every SKU must match `^SKU-[A-Z0-9]{8}$` (the literal prefix `SKU-` followed by exactly 8 uppercase alphanumeric characters).
2. **SKU uniqueness** — creation is rejected if the SKU is already present, checked via `existsBySku` before the insert.
3. **SKU immutability** — an update that carries a different SKU than the stored one is rejected. All other fields are updatable.
4. **Field validation** — `@NotBlank`, `@Positive`, `@Min`, `@Size`, and `@NotNull` on the entity are enforced on every `POST` and `PUT` via `@Valid`.

---

## Error Handling

`GlobalExceptionHandler` (`@RestControllerAdvice`) maps every failure to a predictable status code and JSON body.

| Exception | Status | Response shape |
|---|---|---|
| `MethodArgumentNotValidException` | `400 Bad Request` | Map of field name → message |
| `InvalidSkuFormatException` | `400 Bad Request` | `{"error": "..."}` |
| `ProductNotFoundException` | `404 Not Found` | `{"error": "..."}` |
| `SkuAlreadyExistsException` | `409 Conflict` | `{"error": "..."}` |

### Validation failure — `400`

Request:

```json
{ "name": "", "sku": "SKU-99999999", "price": -10.50, "quantity": 5, "status": "ACTIVE" }
```

Response:

```json
{
  "name": "Product name must not be blank",
  "price": "Price must be a positive number"
}
```

### Invalid SKU format — `400`

```json
{ "error": "SKU must start with SKU- followed by 8 alphanumeric characters." }
```

### Duplicate SKU — `409`

```json
{ "error": "SKU SKU-A1B2C3D4 already exists." }
```

### Product not found — `404`

```json
{ "error": "Product not found with ID: 99" }
```

---

## Repository Queries

`ProductRepository` extends `JpaRepository<Product, Long>` and demonstrates both derived and JPQL queries.

**Derived queries**

| Method | Purpose |
|---|---|
| `existsBySku(String)` | Uniqueness check before insert |
| `findByStatus(ProductStatus)` | Filter by lifecycle status |
| `findByNameContainingIgnoreCase(String)` | Case-insensitive name search |
| `findByQuantityLessThan(Integer)` | Low-stock lookup |

**JPQL queries**

| Method | Query |
|---|---|
| `findByPriceRange(min, max)` | Products within a price band |
| `findAvailableByStatus(status)` | Products with a given status and stock on hand |

Note: the derived and JPQL finders beyond `existsBySku` are implemented at the repository layer but not yet exposed through the controller — they are wired for the search and filtering endpoints planned below.

---

## Project Structure

```
src/main/java/com/assignment14/ProductInventoryAPI/
├── ProductInventoryApiApplication.java   # Entry point
├── controller/
│   └── ProductController.java            # REST endpoints
├── service/
│   └── ProductService.java               # Business rules
├── repository/
│   └── ProductRepository.java            # Data access
├── entity/
│   ├── Product.java                      # JPA entity + validation
│   └── ProductStatus.java                # Status enum
└── exceptions/
    ├── GlobalExceptionHandler.java       # @RestControllerAdvice
    ├── InvalidSkuFormatException.java
    ├── ProductNotFoundException.java
    └── SkuAlreadyExistsException.java

src/main/resources/
└── application.properties                # H2 + JPA configuration
```

The layering is deliberate: the controller handles HTTP concerns only, the service owns business rules, and the repository owns persistence. Constructor injection is used throughout via Lombok's `@RequiredArgsConstructor`, keeping dependencies final and the classes straightforward to unit test.

---

## Testing

```bash
./mvnw test
```

Sample request payloads for Postman — including the success, validation-failure, invalid-SKU, and duplicate-SKU cases, plus ten seed products — are in [`data_examples_for_postman`](src/main/java/com/assignment14/ProductInventoryAPI/data_examples_for_postman).

SQL snippets for verifying persisted state directly in the H2 console are in [`README.md`](src/main/java/com/assignment14/ProductInventoryAPI/README.md) inside the package directory.

---

## Roadmap

- Expose the existing repository finders as query endpoints (`GET /api/products?status=&minPrice=&maxPrice=`)
- Introduce request/response DTOs so the JPA entity is not the API contract
- Pagination and sorting on the list endpoint
- Replace `Double` with `BigDecimal` for monetary values
- Unit tests for `ProductService` and slice tests for `ProductController`
- OpenAPI documentation via springdoc
- Externalize database configuration for a PostgreSQL profile

---

## License

Released for educational purposes as part of the Ostad Spring Boot curriculum.
