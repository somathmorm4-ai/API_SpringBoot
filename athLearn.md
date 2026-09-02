# Learning Path — Spring Boot POS System

A step-by-step guide to learn Spring Boot by studying this POS (Point of Sale) project, from foundations to advanced concepts.

---

## Phase 1: Java & Spring Boot Foundations

### 1.1 Java 21 Core Concepts
- **OOP basics**: Classes, interfaces, inheritance, polymorphism
- **Java 21 features**: Records, sealed classes, pattern matching
- **Annotations**: What they are and how `@Override`, `@Deprecated` work
- **Generics**: Understanding `JpaRepository<T, ID>` type parameters
- **BigDecimal vs double**: Why money uses `BigDecimal` (`Product.price`, `OrderDetail.unit_price`)

### 1.2 Spring Boot Introduction
- What is Spring Boot and why it matters
- **Spring Initializr** and project setup
- Understanding `pom.xml` and Maven dependencies
  - Read `pom.xml:32-109` — analyze each dependency and its purpose
- The `@SpringBootApplication` annotation (`PosSysApplication.java`)
- How `application.properties` configures the app (`src/main/resources/application.properties`)

### 1.3 Maven & Build Tools
- Running the app: `./mvnw spring-boot:run` (port 8088)
- Compiling: `./mvnw clean compile`
- Testing: `./mvnw test`
- Understanding Maven profiles and dependency scopes (`runtime`, `test`, `optional`)

---

## Phase 2: REST API Development

### 2.1 HTTP & REST Basics
- HTTP methods: `GET`, `POST`, `PUT`, `DELETE`
- Status codes: 200, 201, 400, 401, 404, 500
- RESTful URL design (`/api/products`, `/api/orders/{id}`)

### 2.2 Building Controllers — Start Simple
**Start here** — the simplest controller in the project:
- `CashierController.java` — Raw JDBC with `Map<String, Object>`
  - `@RestController`, `@RequestMapping`, `@GetMapping`, `@PostMapping`
  - `@RequestBody` with raw `Map` (no entities, no DTOs)
  - `JdbcTemplate.queryForList()` and `JdbcTemplate.update()`
  - `BeanPropertyRowMapper` for mapping rows to Maps

### 2.3 Controllers with JPA Entities
- `TableController.java` — JPA entities used directly
  - `@Autowired` repository injection
  - `@Valid` on request body for validation
  - Entity as both input and output (no DTO layer yet)

### 2.4 Controllers with DTOs & Mappers
- `CategoryController.java` — raw JDBC (compare with CashierController)
- `ProductController.java` — the full pattern:
  - `ProductRequestDTO` → `ProductMapper.toEntity()` → `ProductRepository.save()`
  - `ProductRepository.findById()` → `ProductMapper.toResponse()` → `ProductResponseDTO`
  - `EntityNotFoundException` handling with `@ExceptionHandler`

---

## Phase 3: Database & JPA

### 3.1 MySQL Setup
- Database: `db_11_1` on `localhost:3308`
- Table naming convention: `tb_*` prefix (`tb_products`, `tb_orders`, etc.)
- Why `spring.jpa.hibernate.ddl-auto=none` — schema managed manually
- Understanding the `Long` vs `INT` id mismatch issue

### 3.2 JPA Entities & Annotations
- `@Entity`, `@Table(name = "tb_...")`
- `@Id`, `@GeneratedValue(strategy = GenerationType.IDENTITY)`
- `@Column(name = "snake_case")` — this project uses snake_case field names
- `@ManyToOne(fetch = FetchType.LAZY)` — Product → Category, Order → Table
- `@ToString.Exclude` and `@EqualsAndHashCode.Exclude` on relationships
- Cascade types on `OrderDetail.order` (cascade delete)

### 3.3 Spring Data JPA Repositories
- `JpaRepository<Entity, Long>` — what it provides out of the box
- Custom queries: `OrderDetailRepository.findByOrderId(Long orderId)` (derived query)
- Custom queries: `UserRepository.findByEmail(String email)`
- `existsByEmail()` pattern

### 3.4 Two Data Access Styles
Study how this project coexists:
| Style | Where Used | Pattern |
|-------|-----------|---------|
| `JdbcTemplate` | CashierController, CategoryController | Raw SQL, `queryForList()`, `update()` |
| Spring Data JPA | Product, Table, Order, User | Repository interface, entity mapping |

---

## Phase 4: DTOs, Mappers & Validation

### 4.1 Data Transfer Objects (DTOs)
- Why DTOs? (hide internal fields, shape response for clients)
- Request DTOs: `ProductRequestDTO`, `OrderRequestDTO`, `RegisterRequestDTO`
- Response DTOs: `ProductResponseDTO`, `OrderResponseDTO`, `RegisterResponseDTO`
- Nested DTOs: `OrderDetailRequestDTO` inside `OrderRequestDTO`

### 4.2 Mapper Components
- `ProductMapper.java` — entity ↔ DTO conversion
  - `toEntity()`: DTO → Entity (resolves `category_id` → `Category` entity via repository)
  - `toResponse()`: Entity → DTO (flattens `Category` to `category_id` + `category_name`)
- `OrderMapper.java` — the most complex mapper
  - Resolves `table_id` → `Table` entity
  - Resolves `product_id` → `Product` entity for each detail line
  - Computes `total = unit_price * qty * (1 - discount_percent / 100)`
  - Computes `order_total` as sum of all detail totals

### 4.3 Bean Validation (Jakarta)
- `@NotBlank`, `@NotNull`, `@Size`, `@Email`
- `@Digits(integer = 12, fraction = 2)` for BigDecimal
- `@DecimalMin("0.01")` for price validation
- `@Valid` on controller method parameters
- Read `ProductRequestDTO.java` and `OrderDetailRequestDTO.java` for examples

---

## Phase 5: Service Layer & Business Logic

### 5.1 Service Layer Pattern
- `@Service` annotation
- Separating business logic from controllers
- `OrderService.java` — full CRUD + business rules
  - `create()`: validates products exist, saves order then details
  - `getAll()`: enriches orders with details and computed totals
  - `checkout()`: sets `time_out` and `payment_method`

### 5.2 User Service & Authentication
- `UserService.java`:
  - Registration: check uniqueness → hash password → save → generate JWT
  - Login: find by email → verify password → generate JWT
  - Profile: extract email from token → return user info

---

## Phase 6: Security — JWT Authentication

### 6.1 JWT (JSON Web Tokens)
- What JWT is and why it's used
- Structure: header, payload, signature
- `JwtService.java`:
  - `generateToken(email, role)` — creates JWT with claims
  - `extractEmail(token)`, `extractRole(token)` — parsing
  - `isValid(token)` — verification
- `jwt.secret` and `jwt.expiration` (24h) in `application.properties`

### 6.2 HTTP Interceptor (Custom Security)
- `AuthInterceptor.java` — implements `HandlerInterceptor`
  - `preHandle()`: checks `Authorization: Bearer <token>` header
  - Allows `OPTIONS` requests (CORS preflight)
  - Returns 401 JSON response on invalid/missing token
- `WebConfig.java` — registers interceptor on `/api/**`
  - Excludes `/api/users/register` and `/api/users/login`
- Understanding this is NOT Spring Security filter chain — it's a simpler approach

### 6.3 Password Security
- `spring-security-crypto` dependency (BCrypt)
- Hashing passwords before storing (`UserService.register()`)
- Verifying passwords on login (`UserService.login()`)

---

## Phase 7: Advanced Topics

### 7.1 File Upload
- `FileController.java` — multipart file upload
- `@RequestParam("file") MultipartFile`
- UUID-based filename generation
- Serving files via `WebConfig` resource handler (`/uploads/**`)
- Max file size configuration in `application.properties`

### 7.2 CORS Configuration
- `WebConfig.java` — `addCorsMappings()`
- Allowing specific origins (`127.0.0.1:5500`, `localhost:5500`)
- Allowed methods and headers
- Why CORS matters for frontend development

### 7.3 API Documentation (Swagger/OpenAPI)
- `springdoc-openapi` dependency
- `OpenApiConfig.java` — title, version, security scheme
- Access at `/swagger-ui.html` or `/v3/api-docs`
- Bearer token security configuration for testing

### 7.4 Lombok
- `@Data` — generates getters, setters, `toString`, `equals`, `hashCode`
- `@ToString.Exclude` — prevents infinite recursion on JPA relationships
- `@EqualsAndHashCode.Exclude` — same for equals/hashCode
- Annotation processing configured in `pom.xml` (maven-compiler-plugin)

---

## Phase 8: Order & Checkout Flow (Complete Feature Study)

### 8.1 End-to-End Order Flow
Study the complete lifecycle:
1. **Create Order** — `POST /api/orders`
   - Client sends: `table_id`, `cashier_id`, `queue_no`, `order_details[]`
   - Server: validates products, computes totals, saves order + details
2. **View Orders** — `GET /api/orders`
   - Server: loads orders with JPA lazy loading, maps to DTOs with names
3. **Checkout** — `PUT /api/orders/{id}/checkout`
   - Server: sets `time_out = now()`, saves `payment_method`

### 8.2 Entity Relationship Diagram
```
tb_categories (1) ──── (N) tb_products
tb_table (1) ──── (N) tb_orders
tb_orders (1) ──── (N) tb_order_details
tb_products (1) ──── (N) tb_order_details
tb_users (standalone)
tb_cashiers (standalone, SQL only)
```

### 8.3 Total Computation Formula
```
detail_total = unit_price × qty × (1 - discount_percent / 100)
order_total  = Σ detail_total (for all details in an order)
```

---

## Phase 9: Testing

### 9.1 Current State
- Only `PosSysApplicationTests.java` exists (context-load test)
- Run: `./mvnw test`

### 9.2 What to Add Next
- **Unit tests**: Test mappers, services in isolation (mock repositories)
- **Integration tests**: Test controllers with `@WebMvcTest` + `MockMvc`
- **Repository tests**: `@DataJpaTest` for repository query verification
- Test the JWT interceptor: mock tokens, verify 401 responses

---

## Phase 10: Practice Exercises

### Beginner
1. Add a new field to `Product` (e.g., `description`) with migration
2. Create a new controller following the `CashierController` pattern (raw JDBC)
3. Add validation to `CashierController` inputs

### Intermediate
4. Convert `CategoryController` from JdbcTemplate to JPA repository style
5. Add pagination to `GET /api/products` (`Pageable`, `Page<Product>`)
6. Create a `GET /api/reports/daily-sales` endpoint that sums orders by date

### Advanced
7. Add Spring Security filter chain instead of the custom interceptor
8. Implement role-based access (admin vs cashier) using the `role` field
9. Add Swagger `@Operation` and `@Schema` annotations to all controllers
10. Write unit tests for `OrderMapper` and `OrderService`

---

## Recommended Study Order

```
Week 1: Phase 1-2 (Foundations + Controllers)
Week 2: Phase 3-4 (Database + DTOs)
Week 3: Phase 5-6 (Services + Security)
Week 4: Phase 7-8 (Advanced + Order Flow)
Week 5: Phase 9-10 (Testing + Exercises)
```

---

## Key Files Reference

| Concept | File | Lines to Study |
|---------|------|----------------|
| Entry point | `PosSysApplication.java` | All |
| Simplest controller | `CashierController.java` | All |
| JPA + DTOs | `ProductController.java` | All |
| Full business logic | `OrderController.java` + `OrderService.java` | All |
| Entity mapping | `OrderMapper.java` | All |
| JWT auth | `JwtService.java` + `AuthInterceptor.java` | All |
| Config | `WebConfig.java` | All |
| Validation | `ProductRequestDTO.java`, `OrderDetailRequestDTO.java` | All |
| Relationships | `Order.java`, `OrderDetail.java`, `Product.java` | All |
| Application config | `application.properties` | All |
