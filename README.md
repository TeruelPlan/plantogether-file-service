# File Service

> Service de gestion des fichiers avec MinIO (S3-compatible)

## Rôle dans l'architecture

Le File Service gère tous les fichiers uploadés dans PlanTogether. Il génère des URLs présignées sécurisées pour que les
autres microservices puissent uploader et télécharger directement via MinIO, sans passer par ce service (architecture
efficace). Cela inclut : images de couverture de voyage, photos de destinations, reçus de dépenses, images de chat et
avatars utilisateur. Toutes les URLs sont temporaires et révoquées après expiration.

## Fonctionnalités

- Génération d'URLs présignées pour upload sécurisé (PUT)
- Génération d'URLs présignées pour téléchargement (GET)
- Suppression de fichiers avec validation de propriété
- Organisation des fichiers par voyage et catégorie
- Validation de type et taille de fichier
- Support multi-tenant (isolation par voyage)
- Stockage S3-compatible via MinIO
- Audit des accès (logs)
- Cleanup automatique des fichiers orphelins

## Endpoints REST

| Méthode | Endpoint                              | Description                                 |
|---------|---------------------------------------|---------------------------------------------|
| POST    | `/api/files/presigned-upload`         | Générer une URL présignée d'upload          |
| GET     | `/api/files/presigned-download/{key}` | Générer une URL présignée de téléchargement |
| DELETE  | `/api/files/{key}`                    | Supprimer un fichier                        |
| HEAD    | `/api/files/{key}`                    | Vérifier l'existence d'un fichier           |
| GET     | `/api/files/{tripId}/list`            | Lister les fichiers d'un voyage             |

## Modèle de données

**FileMetadata**

- `id` (UUID) : identifiant unique
- `file_key` (String) : chemin complet dans MinIO
- `trip_id` (UUID) : voyage propriétaire
- `category` (ENUM: TRIP_COVER, DESTINATION_PHOTO, EXPENSE_RECEIPT, CHAT_IMAGE, USER_AVATAR) : catégorie
- `original_filename` (String) : nom d'origine (sans utiliser en MinIO)
- `content_type` (String) : MIME type (image/jpeg, etc.)
- `file_size` (Long) : taille en bytes
- `uploaded_by` (UUID) : ID Keycloak de l'uploader
- `created_at` (Timestamp)
- `expires_at` (Timestamp, nullable) : date d'expiration pour cleanup
- `access_count` (Integer, default: 0)

Les métadonnées sont stockées en PostgreSQL pour tracking et audit.

## Endpoints - Détails

### POST /api/files/presigned-upload

**Request Body :**

```json
{
  "tripId": "uuid-voyage",
  "category": "EXPENSE_RECEIPT",
  "filename": "facture-hotel.pdf",
  "contentType": "application/pdf",
  "fileSizeBytes": 524288
}
```

**Response :**

```json
{
  "presignedUrl": "http://minio:9000/plantogether/trips/uuid.../EXPENSE_RECEIPT/...?X-Amz-...",
  "key": "trips/uuid-voyage/EXPENSE_RECEIPT/xxxxxx-facture-hotel.pdf",
  "expiresIn": 3600
}
```

### GET /api/files/presigned-download/{key}

**Response :**

```json
{
  "presignedUrl": "http://minio:9000/plantogether/...",
  "expiresIn": 3600
}
```

### DELETE /api/files/{key}

Suppression sécurisée (validation : l'utilisateur doit être le créateur ou organisateur du voyage).

## Structure des fichiers dans MinIO

```
plantogether/
├── trips/
│   ├── {tripId}/
│   │   ├── TRIP_COVER/
│   │   │   └── {uuid}-{filename}
│   │   ├── DESTINATION_PHOTO/
│   │   │   └── {uuid}-{filename}
│   │   ├── EXPENSE_RECEIPT/
│   │   │   └── {uuid}-{filename}
│   │   ├── CHAT_IMAGE/
│   │   │   └── {uuid}-{filename}
│   │   └── USER_AVATAR/
│   │       └── {uuid}-{filename}
│   └── ...
```

**Conventions :**

- Clé unique : UUID + nom original pour éviter les collisions
- Organisation par catégorie pour gestion aisée
- Pas d'espaces ni caractères spéciaux dans les clés

## Configuration

```yaml
server:
  port: 8088
  servlet:
    context-path: /
    
spring:
  application:
    name: plantogether-file-service
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  datasource:
    url: jdbc:postgresql://postgres:5432/plantogether_file
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  redis:
    host: ${REDIS_HOST:redis}
    port: ${REDIS_PORT:6379}

keycloak:
  serverUrl: ${KEYCLOAK_SERVER_URL:http://keycloak:8080}
  realm: ${KEYCLOAK_REALM:plantogether}
  clientId: ${KEYCLOAK_CLIENT_ID}

minio:
  endpoint: ${MINIO_ENDPOINT:http://minio:9000}
  accessKey: ${MINIO_ACCESS_KEY}
  secretKey: ${MINIO_SECRET_KEY}
  bucket: ${MINIO_BUCKET:plantogether}
  
file:
  presignedUrl:
    expirationSeconds: 3600  # 1 heure
  validation:
    maxFileSizeBytes: 10485760  # 10 MB
    allowedContentTypes:
      - image/jpeg
      - image/png
      - image/webp
      - image/gif
      - application/pdf
      - text/plain
    allowedCategories:
      - TRIP_COVER
      - DESTINATION_PHOTO
      - EXPENSE_RECEIPT
      - CHAT_IMAGE
      - USER_AVATAR
  cleanup:
    enabled: true
    orphanedFileAgeHours: 24
    expiredFileAgeHours: 1
    scheduleExpression: "0 0 2 * * *"  # 2h du matin chaque jour
```

## Lancer en local

```bash
# Prérequis : Docker Compose (infra), Java 21+, Maven 3.9+

# Option 1 : Maven
mvn spring-boot:run

# Option 2 : Docker
docker build -t plantogether-file-service .
docker run -p 8088:8081 \
  -e KEYCLOAK_SERVER_URL=http://host.docker.internal:8080 \
  -e DB_USER=postgres \
  -e DB_PASSWORD=postgres \
  -e MINIO_ENDPOINT=http://host.docker.internal:9000 \
  -e MINIO_ACCESS_KEY=minioadmin \
  -e MINIO_SECRET_KEY=minioadmin \
  plantogether-file-service
```

## Dépendances

- **Keycloak 24+** : authentification et autorisation
- **PostgreSQL 16** : métadonnées des fichiers
- **MinIO** : stockage S3-compatible
- **Redis** : cache des métadonnées
- **Spring Boot 3.3.6** : framework web
- **AWS SDK S3** (via MinIO client) : interaction avec MinIO
- **Spring Cloud Netflix Eureka** : service discovery

## Workflow d'upload

```
Client/Service                File Service              MinIO
    |                              |                      |
    +---POST /api/files/presigned-upload--->|              |
    |                              |                      |
    |                    (Validation + création métadonnées) |
    |                              |                      |
    |<--presignedUrl (valide 1h)---+                      |
    |                              |                      |
    +--PUT presignedUrl + fichier--+---------------------->|
    |                              |        (directement)  |
    |                              |     (pas via service)|
    |<--200 OK (ou erreur)--+------+-------MinIO          |
    |                              |                      |
```

## Workflow de téléchargement

```
Client                    File Service              MinIO
    |                          |                     |
    +---GET /api/files/presigned-download/{key}--->|
    |                          |                     |
    |                  (Validation de permissions)   |
    |                          |                     |
    |<--presignedUrl (valide 1h)--+                 |
    |                          |                     |
    +--GET presignedUrl--------+-------------------->|
    |                          |                     |
    |<--fichier (directement MinIO, pas via service)|
    |                          |                     |
```

## Validation

**Tailles de fichier par catégorie :**

- TRIP_COVER : max 5 MB
- DESTINATION_PHOTO : max 5 MB
- EXPENSE_RECEIPT : max 10 MB
- CHAT_IMAGE : max 5 MB
- USER_AVATAR : max 2 MB

**Content types acceptés :**

- Images : JPEG, PNG, WebP, GIF
- Documents : PDF
- Texte : TXT

**Autres validations :**

- Pas de fichiers exécutables
- Pas de scripts
- Vérification du magic number (vraie extension)

## Cleanup automatique

Un job planifié recherche et supprime :

1. Fichiers orphelins : uploadés mais non associés à une entité après 24h
2. Fichiers expirés : avec `expires_at` dépassée

## Notes de sécurité

- URLs présignées sont temporaires (1 heure par défaut)
- Validation stricte des content-types
- Scan antivirus recommandé à la production
- Zéro PII : pas de données utilisateur dans les fichiers
- Chiffrement MinIO recommandé en production
- Isolation par voyage (zéro accès cross-voyage)
- Logs d'audit complets des uploads/téléchargements/suppressions

## Performance

- Pas de streaming via File Service (presigned URLs)
- Requêtes directes MinIO pour upload/download
- Métadonnées en cache Redis (TTL 1h)
- Scalabilité horizontale (stateless)

## Exemple d'utilisation (client Java)

```java
// 1. Demander une URL présignée
FileUploadRequest req = new FileUploadRequest(
  "trip-uuid",
  "EXPENSE_RECEIPT",
  "reçu.pdf",
  "application/pdf",
  524288L
);
FileUploadResponse response = fileService.getPresignedUploadUrl(req);

// 2. Upload direct vers MinIO
uploadToPresignedUrl(response.getPresignedUrl(), file);

// 3. Plus tard : télécharger
String downloadUrl = fileService.getPresignedDownloadUrl("trips/trip-uuid/EXPENSE_RECEIPT/...");
```
