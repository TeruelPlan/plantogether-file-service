# File Service

> File management service via MinIO presigned URLs (S3-compatible)

## Role in the Architecture

The File Service generates presigned URLs for uploading and downloading files in MinIO. **No file
passes through this service** — it acts as a signed URL generation proxy. Other services can also
call it via gRPC to obtain read URLs (e.g. receipt photos).

## Features

- Presigned PUT URL generation for direct upload to MinIO (time-limited)
- Presigned GET URL generation for secure reading
- Device identity verification before any URL generation
- gRPC server (`GetPresignedUrl`) consumed by other services

## REST Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/files/presigned-upload` | Get a presigned PUT URL (direct upload to MinIO) |
| GET | `/api/v1/files/{key}` | Get a presigned GET URL (read) |

## gRPC Server (port 9088)

The File Service exposes a gRPC server consumed by other microservices:

| RPC | Description |
|-----|-------------|
| `GetPresignedUrl(key)` | Generates a presigned GET URL for the given key |

## Upload Flow

1. The Flutter client calls `POST /api/v1/files/presigned-upload` with the file name and type
2. The File Service generates a unique key (`trip/{tripId}/{uuid}.{ext}`) and a presigned PUT URL (15 min validity)
3. Flutter uploads directly to MinIO via the presigned PUT URL (without going through the API)
4. Flutter sends the key (e.g. `trip/abc/photo.jpg`) to the relevant service (trip-service, expense-service, etc.)
5. For reading, the service calls `GetPresignedUrl` via gRPC or Flutter calls `GET /api/v1/files/{key}`

## Data Model

This service has **no database** — it is stateless. MinIO keys are managed directly by business services
(stored in their own tables: `cover_image_key`, `receipt_key`, `image_key`).

## Configuration

```yaml
server:
  port: 8088

spring:
  application:
    name: plantogether-file-service

minio:
  endpoint: ${MINIO_ENDPOINT:http://minio:9000}
  access-key: ${MINIO_ACCESS_KEY:minioadmin}
  secret-key: ${MINIO_SECRET_KEY:minioadmin}
  bucket: ${MINIO_BUCKET:plantogether}
  presigned-url-expiry: 900  # 15 minutes

grpc:
  server:
    port: 9088
```

## Running Locally

```bash
# Prerequisites: docker compose up -d (MinIO included)
# + plantogether-proto and plantogether-common installed

mvn spring-boot:run
```

## Dependencies

- **MinIO**: S3-compatible object storage (presigned URLs)
- **plantogether-proto**: gRPC contracts (server exposed on 9088)
- **plantogether-common**: DeviceIdFilter, SecurityAutoConfiguration, CorsConfig

## Rate Limiting

| Rule | Limit |
|------|-------|
| `POST /api/v1/files/presigned-upload` | 20 uploads / hour / device |

## Security

- Anonymous device-based identity: `X-Device-Id` header on every request
- `DeviceIdFilter` (from plantogether-common, auto-configured via `SecurityAutoConfiguration`) extracts the device UUID and sets the SecurityContext principal
- No JWT, no Keycloak, no login, no sessions
- Presigned URLs expire after 15 minutes (upload) / configurable duration (read)
- MinIO keys follow a structured pattern (`trip/{tripId}/{uuid}.{ext}`) to avoid collisions
- No user data is stored in this service
