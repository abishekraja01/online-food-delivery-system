# Project Architecture & Coding Standards

## 1. Core Stack & Versions
- **Runtime & Framework:** Java 21 (LTS), Spring Boot 4.2.0, Spring Cloud 2026.0.0.
- **Data & Persistence:** Spring Data JPA, Hibernate, MySQL (`mysql-connector-j`).
- **Utilities & Docs:** Lombok, Springdoc OpenAPI 3.0.0 (`springdoc-openapi-starter-webmvc-ui`).
- **Build System:** Multi-module Maven project (`online-food-delivery-system` parent POM). Child POMs inherit versions from `../pom.xml` and must not redeclare `<java.version>` or duplicate `<dependencyManagement>`.

## 2. Spring Boot 4.x Specific Conventions
- **Jackson 3 Packages:** Spring Boot 4 uses Jackson 3 (`tools.jackson.databind.ObjectMapper`, `tools.jackson.core.*`). Do NOT import `com.fasterxml.jackson.databind.*` unless explicitly required by a third-party library. Jackson annotations (`@JsonProperty`, `@JsonIgnore`, `@JsonInclude`) remain under `com.fasterxml.jackson.annotation.*`.
- **API Gateway Starter:** Use `spring-cloud-starter-gateway-server-webmvc` (servlet-based MVC gateway) and configure routes under `spring.cloud.gateway.server.webmvc.*`.
- **Null-Safety:** Prefer JSpecify annotations (`org.jspecify.annotations.Nullable`, `org.jspecify.annotations.NonNull`) over legacy `javax.annotation` or Spring-specific nullability annotations.

## 3. Microservices & Database Boundaries
- **Services & Schemas:**
    - `eurekaserver`: Netflix Eureka service registry (no database).
    - `api-gateway`: Public entry point, JWT validation filter, CORS, Swagger aggregation (no database).
    - `authentication-microservice`: Auth, signup, JWT issuance -> MySQL schema `authentication`.
    - `user-microservice`: User profiles -> MySQL schema `userdb`.
    - `restaurant-microservice`: Restaurants, menus, delivery agents -> MySQL schema `restaurant`.
    - `cart-microservice`: Shopping carts & cart items -> MySQL schema `cartdb`.
    - `order-transaction-microservice`: Orders, order items, payments/transactions -> MySQL schema `ordertransactiondb`.
- **Strict Isolation:** Never share JPA `@Entity` classes, repositories, or database tables across services. All cross-service communication must go through Spring Cloud OpenFeign (`@FeignClient`) using Eureka service IDs.

## 4. Package Structure & Layering (Per Domain Service)
Organize code under `com.fooddelivery.<servicename>` using clean layers:
- `controller/` — REST endpoints (`@RestController`), request validation (`@Valid`), HTTP status mapping via `ResponseEntity`.
- `service/` — Business logic interfaces and implementations (`@Service`, `@Transactional`).
- `repository/` — Spring Data JPA interfaces (`JpaRepository`).
- `entity/` — JPA `@Entity` classes (internal to the service only).
- `dto/` — Immutable Java `record` types for API requests and responses. Never expose `@Entity` objects in controller signatures.
- `client/` — `@FeignClient` interfaces and fallback handlers for external service calls.
- `exception/` — Custom domain exceptions and a centralized `@RestControllerAdvice`.
- `config/` — Security, OpenFeign interceptors, OpenAPI, and CORS configuration.

## 5. Coding & Error-Handling Standards
- **Dependency Injection:** Always use constructor injection (`@RequiredArgsConstructor` for services/controllers). Never use field-level `@Autowired`.
- **Standardized Error Responses:** Every service must implement a `@RestControllerAdvice` returning a consistent `ErrorResponse` record (`LocalDateTime timestamp`, `int status`, `String error`, `String message`, `String path`).
- **Configuration & Secrets:** External Config Server is intentionally removed. Keep all runtime configuration in each service's `src/main/resources/application.properties`. Never hardcode passwords or JWT secrets—always use placeholders with local defaults (e.g., `${DB_PASSWORD:root}`, `${JWT_SECRET:local-dev-secret-key-at-least-32-chars}`).