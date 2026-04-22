# CLAUDE.md

This file provides guidance to Claude when working with code in this repository.

## Commands

```bash
# Build
mvn clean package

# Build without tests
mvn clean package -DskipTests

# Run locally
mvn spring-boot:run

# Run all tests
mvn test

# Run a single test class
mvn test -Dtest=ClassName

# Run a single test method
mvn test -Dtest=ClassName#methodName

# Docker build
docker build -t plantogether-file-service .
```

**Prerequisites:**

Local Maven builds resolve `plantogether-parent`, `plantogether-bom`, `plantogether-common`, and `plantogether-proto`
from GitHub Packages. Export a PAT with `read:packages` scope before running any `mvn` command:

```bash
export GITHUB_ACTOR=<your-github-username>
export GITHUB_TOKEN=<your-PAT-with-read:packages>
mvn -s .settings.xml clean package
```

## Architecture

Spring Boot 3.5.9 microservice (Java 21). Manages file storage via MinIO presigned URLs. **Files never pass
through this service** — the service only generates short-lived signed URLs for clients to upload/download
directly to/from MinIO.

**Ports:** REST `8088` · gRPC `9088`

**Package:** `com.plantogether.file`

### Package structure

```
com.plantogether.file/
├── config/          # MinioConfig
├── controller/      # REST controllers
├── service/         # PresignedUrlService
├── dto/             # Request/Response DTOs (Lombok @Data @Builder)
└── grpc/
    └── server/      # FileGrpcServiceImpl (exposes gRPC on port 9088)
```

No database — this service does not persist any data.

### Infrastructure dependencies

| Dependency | Default (local) | Purpose |
|---|---|---|
| MinIO | `localhost:9000` | Actual file storage (S3-compatible, via AWS SDK v2) |


### gRPC server (port 9088)

Implements `FileGrpcService` (defined in `plantogether-proto`):

| Method | Description |
|---|---|
| `GetPresignedUrl(key, operation)` | Returns a short-lived presigned MinIO URL for `UPLOAD` or `DOWNLOAD` |

Called by all other microservices that need to generate upload/download URLs for files.

### MinIO key structure

`trips/{tripId}/{CATEGORY}/{uuid}-{filename}`

File categories: `TRIP_COVER`, `DESTINATION_PHOTO`, `EXPENSE_RECEIPT`, `CHAT_IMAGE`, `USER_AVATAR`.

Presigned URL TTL: 1h for upload, 1h for download.

### REST API (`/api/v1/files/`)

| Method | Endpoint | Notes |
|---|---|---|
| POST | `/api/v1/files/presigned-upload` | Get a presigned PUT URL — body: `{ key, contentType }` |
| GET | `/api/v1/files/{key}` | Get a presigned GET URL for reading |

Flutter calls these endpoints to get the URL, then uploads/downloads directly to MinIO without going through
this service again.

### Security

- Anonymous device-based identity via `DeviceIdFilter` (from `plantogether-common`, auto-configured via `SecurityAutoConfiguration`)
- `X-Device-Id` header extracted and set as SecurityContext principal
- No JWT, no Keycloak, no login, no sessions
- No SecurityConfig.java needed — `SecurityAutoConfiguration` handles everything
- Public endpoints: `/actuator/health`, `/actuator/info`
- Rate limiting: 20 presigned upload requests per device per hour (Bucket4j + Redis)

### Environment variables

| Variable | Default | Description |
|---|---|---|
| `MINIO_ENDPOINT` | — | MinIO endpoint URL (e.g. `http://localhost:9000`) |
| `MINIO_ACCESS_KEY` | — | MinIO access key |
| `MINIO_SECRET_KEY` | — | MinIO secret key |
| `MINIO_BUCKET` | `plantogether` | Target bucket name |
