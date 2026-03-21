# File Service

> Service de gestion des fichiers via presigned URLs MinIO (S3-compatible)

## Rôle dans l'architecture

Le File Service génère des presigned URLs pour les uploads et downloads de fichiers dans MinIO. **Aucun fichier
ne transite par ce service** : il agit comme un proxy de génération d'URLs signées. Les autres services peuvent
également l'appeler via gRPC pour obtenir des URLs de lecture (ex. photo de justificatif).

## Fonctionnalités

- Génération de presigned PUT URLs pour l'upload direct vers MinIO (durée limitée)
- Génération de presigned GET URLs pour la lecture sécurisée
- Vérification JWT avant toute génération d'URL
- Exposition d'un serveur gRPC (`GetPresignedUrl`) consommé par les autres services

## Endpoints REST

| Méthode | Endpoint | Description |
|---------|----------|-------------|
| POST | `/api/v1/files/presigned-upload` | Obtenir un presigned PUT URL (upload direct vers MinIO) |
| GET | `/api/v1/files/{key}` | Obtenir un presigned GET URL (lecture) |

## gRPC Server (port 9088)

Le File Service expose un serveur gRPC consommé par les autres microservices :

| RPC | Description |
|-----|-------------|
| `GetPresignedUrl(key)` | Génère un presigned GET URL pour la clé donnée |

## Flux d'upload

1. Le client Flutter appelle `POST /api/v1/files/presigned-upload` avec le nom et type du fichier
2. Le File Service génère une clé unique (`trip/{tripId}/{uuid}.{ext}`) et un presigned PUT URL (validité 15 min)
3. Flutter upload directement vers MinIO via le presigned PUT URL (sans passer par l'API)
4. Flutter transmet la clé (ex. `trip/abc/photo.jpg`) au service concerné (trip-service, expense-service, etc.)
5. Pour la lecture, le service appelle `GetPresignedUrl` via gRPC ou Flutter appelle `GET /api/v1/files/{key}`

## Modèle de données

Ce service ne possède **pas de base de données** — il est sans état. Les clés MinIO sont gérées
directement par les services métier (stockées dans leurs propres tables : `cover_image_key`, `receipt_key`, `image_key`).

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

## Lancer en local

```bash
# Prérequis : docker compose --profile essential up -d (MinIO inclus)
# + plantogether-proto et plantogether-common installés

mvn spring-boot:run
```

## Dépendances

- **Keycloak 24+** : validation JWT (endpoints REST)
- **MinIO** : stockage objet S3-compatible (presigned URLs)
- **plantogether-proto** : contrats gRPC (serveur exposé sur 9088)
- **plantogether-common** : CorsConfig, sécurité partagée

## Rate Limiting

| Règle | Limite |
|-------|--------|
| `POST /api/v1/files/presigned-upload` | 20 uploads / heure / utilisateur |

## Sécurité

- Tous les endpoints REST requièrent un token Bearer Keycloak valide
- Les presigned URLs expirent après 15 minutes (upload) / durée configurable (lecture)
- Les clés MinIO suivent un pattern structuré (`trip/{tripId}/{uuid}.{ext}`) pour éviter les collisions
- Aucune donnée utilisateur n'est stockée dans ce service
