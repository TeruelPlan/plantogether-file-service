# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build
mvn clean package

# Run locally
mvn spring-boot:run

# Run all tests
mvn test

# Run a single test class
mvn test -Dtest=ClassName

# Run a single test method
mvn test -Dtest=ClassName#methodName

# Build Docker image
docker build -t plantogether-file-service .
```

The `plantogether-common` dependency (shared exceptions/DTOs) must be installed in the local Maven repository before building:
```bash
# From the plantogether-common module root:
mvn install -DskipTests
```

## Architecture

This is a Spring Boot 3.3.6 microservice (Java 21) in the PlanTogether platform. It runs on port **8088**.

**Core concept — presigned URLs:** Files never pass through this service. The service generates short-lived presigned URLs (1h default) for clients to upload/download directly to/from MinIO. The service only manages metadata in PostgreSQL and validates permissions.

**Key infrastructure dependencies:**
- **MinIO** (S3-compatible): actual file storage, accessed via AWS SDK S3 v2
- **PostgreSQL** (`plantogether_file` DB): `FileMetadata` table tracking uploads
- **Redis**: caches file metadata (TTL 1h)
- **RabbitMQ**: async event messaging
- **Keycloak**: JWT authentication (realm roles extracted from `realm_access.roles` claim)
- **Eureka**: service discovery

**Security:** `KeycloakJwtConverter` extracts Keycloak realm roles and maps them to `ROLE_<ROLE>` Spring authorities. All endpoints require authentication except `/actuator/health` and `/actuator/info`. Method-level security (`@EnableMethodSecurity`) is enabled.

**Database schema:** Managed by Flyway (`src/main/resources/db/migration/`). `ddl-auto: validate` — Hibernate validates against the schema, never modifies it. Add new migrations as `V{n}__description.sql`.

**MinIO file key structure:** `trips/{tripId}/{CATEGORY}/{uuid}-{filename}`

**File categories:** `TRIP_COVER`, `DESTINATION_PHOTO`, `EXPENSE_RECEIPT`, `CHAT_IMAGE`, `USER_AVATAR`

**Exception handling:** `GlobalExceptionHandler` handles `ResourceNotFoundException` and `AccessDeniedException` from `plantogether-common`, plus `MethodArgumentNotValidException`. All return `ErrorResponse` from the common library.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DB_HOST` | `localhost` | PostgreSQL host |
| `DB_USER` | `plantogether` | DB username |
| `DB_PASSWORD` | `plantogether` | DB password |
| `RABBITMQ_HOST` | `localhost` | RabbitMQ host |
| `REDIS_HOST` | `localhost` | Redis host |
| `KEYCLOAK_URL` | `http://localhost:8180` | Keycloak base URL |
| `EUREKA_URL` | `http://localhost:8761/eureka/` | Eureka server |
| `MINIO_ENDPOINT` | — | MinIO endpoint URL |
| `MINIO_ACCESS_KEY` | — | MinIO access key |
| `MINIO_SECRET_KEY` | — | MinIO secret key |
| `MINIO_BUCKET` | `plantogether` | Target bucket |
