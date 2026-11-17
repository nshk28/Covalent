# StrataFS
# StrataFS

StrataFS is a distributed cloud file storage and sharing platform (inspired by Google Drive) built with **Java and Spring Boot**. It combines scalable object storage, resumable chunked uploads, automated storage optimization, and policy-driven file sharing.

The platform is designed to accept large files reliably over unstable networks, deduplicate and tier stored data to cut cost, and share files through temporary links that enforce their own rules and revoke themselves automatically.

---

## Key Differentiators

### 1. Self-Healing Resumable Uploads
Interrupted large-file uploads recover automatically, even after client disconnects or server restarts.
- Upload session state is persisted in PostgreSQL (and cached in Redis).
- Every chunk is verified with a SHA-256 checksum before it is accepted.
- The client asks "which chunks do you have?" and resumes from the first missing chunk instead of restarting.
- A background sweeper reclaims stale or corrupted sessions.

### 2. Smart Storage Optimizer
- **Deduplication**: content-addressable storage keyed by SHA-256. Identical files share one physical blob (reference counted).
- **Archival recommendations**: flags files not accessed for N days and suggests moving them to a cold tier.
- **Abandoned storage detection**: finds expired upload sessions, orphaned blobs, and long-trashed files.

### 3. Policy-Based Secure Sharing
Temporary share links with configurable policies:
- Expiration time
- Maximum download count
- Recipient restrictions (allow-listed emails, verified through OTP)
- Automatic revocation when a policy expires or is violated

---

## High-Level Architecture (HLD)

```mermaid
flowchart TD
    U[User / Client App] -->|HTTPS| GW[API Gateway - Spring Cloud Gateway]
    GW --> AUTH[Auth Service - JWT / OAuth2]
    GW --> FS[File Service - metadata, folders, trash]
    GW --> UP[Upload Service - resumable chunked uploads]
    GW --> SH[Sharing Service - policy engine]

    UP -->|chunks, multipart| OBJ[(Object Storage - S3 / MinIO)]
    UP -->|session and chunk state| PG[(PostgreSQL)]
    UP -->|hot session cache| REDIS[(Redis)]
    UP -->|UploadCompleted event| MQ{{RabbitMQ}}

    FS --> PG
    FS --> REDIS
    SH --> PG
    SH -->|atomic download counters| REDIS
    SH -->|stream file| OBJ

    MQ --> OPT[Storage Optimizer Worker]
    MQ --> NOTIF[Notification Worker]

    subgraph Background Jobs
        OPT -->|hash, dedup, cold scan, orphan scan| PG
        OPT -->|move to archive tier, delete orphans| OBJ
        SWEEP[Upload Sweeper - Scheduler]
        EXPIRE[Share Expiry Scheduler]
    end

    SWEEP --> PG
    SWEEP --> OBJ
    EXPIRE --> PG
    EXPIRE --> REDIS
    NOTIF -->|email / OTP / alerts| EXT[SMTP / Notification Provider]

    R[Recipient] -->|share link| GW
```

### Component Responsibilities

| Component | Responsibility |
| --- | --- |
| **API Gateway** | Routing, rate limiting, JWT validation, TLS termination |
| **Auth Service** | Registration, login, token issue and refresh, roles |
| **File Service** | Files, folders, versions, trash, search, storage quota |
| **Upload Service** | Initiate, receive, verify, resume, and finalize chunked uploads |
| **Sharing Service** | Create and validate share links, enforce policies, serve downloads |
| **Optimizer Worker** | Dedup, cold-file detection, abandoned storage cleanup |
| **Upload Sweeper** | Expires stale sessions and aborts incomplete multipart uploads |
| **Share Expiry Scheduler** | Revokes expired or exhausted links |
| **Notification Worker** | Sends OTP, share invites, and storage reports |

---

## Core Flows

### Resumable Upload

1. **Initiate**: client sends file name, size, and full-file SHA-256. Server returns `uploadId`, `chunkSize`, and `totalChunks`. If the same `blob_hash` already exists, the server short-circuits (instant upload, no bytes transferred).
2. **Upload chunks**: client sends `PUT /uploads/{id}/chunks/{n}` with a `X-Chunk-Checksum` header. Server recomputes the hash, rejects mismatches, stores the chunk as an S3 multipart part, and records it.
3. **Disconnect or crash**: nothing is lost because chunk state lives in PostgreSQL.
4. **Resume**: client calls `GET /uploads/{id}` and receives the list of received chunks, then uploads only the missing ones.
5. **Complete**: server verifies all chunks are present, completes the multipart upload, validates the full-file hash, creates the file record, and publishes `UploadCompleted`.
6. **Self-healing**: the sweeper aborts sessions idle past the TTL and frees their storage.

### Policy-Based Share Download

1. Owner creates a link with `expiresAt`, `maxDownloads`, and `allowedEmails`.
2. Server stores only the **hash** of the random 256-bit token (the raw token is shown once).
3. Recipient opens the link; if restricted, they verify their email through an OTP.
4. Server validates status, expiry, and the atomic download counter, then streams the file.
5. On expiry, exhaustion, or violation the link flips to `REVOKED` or `EXPIRED` and all further requests are denied.

---

## Low-Level Design (LLD)

### Class Diagram

```mermaid
classDiagram
    class UploadController {
        +initiate(InitUploadRequest) UploadSessionResponse
        +uploadChunk(uploadId, chunkNo, bytes, checksum) ChunkAck
        +getStatus(uploadId) UploadStatusResponse
        +complete(uploadId) FileResponse
        +abort(uploadId) void
    }
    class UploadService {
        +initiate(req) UploadSession
        +acceptChunk(uploadId, chunkNo, data, checksum) ChunkAck
        +getMissingChunks(uploadId) List~Integer~
        +finalizeUpload(uploadId) FileMetadata
    }
    class ChecksumVerifier {
        +sha256(bytes) String
        +verify(bytes, expected) boolean
    }
    class StorageProvider {
        <<interface>>
        +initMultipart(key) String
        +uploadPart(key, uploadToken, partNo, data) String
        +completeMultipart(key, uploadToken, parts) void
        +abortMultipart(key, uploadToken) void
        +openStream(key) InputStream
        +changeTier(key, tier) void
        +delete(key) void
    }
    class S3StorageProvider
    class UploadSession {
        +UUID id
        +UUID ownerId
        +String fileName
        +long size
        +String fileHash
        +int totalChunks
        +UploadStatus status
        +Instant expiresAt
    }
    class UploadChunk {
        +UUID sessionId
        +int chunkNo
        +String checksum
        +String partEtag
    }
    class UploadSweeper {
        +sweepStaleSessions() void
    }

    class ShareController {
        +createLink(CreateShareRequest) ShareLinkResponse
        +download(token) Stream
        +revoke(linkId) void
        +verifyRecipient(token, email, otp) void
    }
    class ShareService {
        +create(fileId, policy) ShareLink
        +validateAndConsume(token, recipient) FileMetadata
        +revoke(linkId) void
    }
    class SharePolicyEvaluator {
        <<interface>>
        +evaluate(link, ctx) PolicyResult
    }
    class ExpiryPolicy
    class DownloadLimitPolicy
    class RecipientPolicy
    class ShareLink {
        +UUID id
        +UUID fileId
        +String tokenHash
        +Instant expiresAt
        +Integer maxDownloads
        +int downloadCount
        +Set~String~ allowedEmails
        +ShareStatus status
    }
    class ShareExpiryScheduler {
        +revokeExpiredLinks() void
    }

    class OptimizerService {
        +findDuplicates() List~DuplicateGroup~
        +recommendArchival(days) List~ArchiveRecommendation~
        +detectAbandoned() List~AbandonedItem~
        +reclaim(items) ReclaimReport
    }
    class DedupService {
        +registerBlob(hash, size, key) Blob
        +incrementRef(hash) void
        +decrementRef(hash) void
    }
    class FileMetadata {
        +UUID id
        +UUID ownerId
        +UUID folderId
        +String name
        +String blobHash
        +Instant lastAccessedAt
        +StorageTier tier
    }
    class Blob {
        +String hash
        +String storageKey
        +long size
        +int refCount
    }

    UploadController --> UploadService
    UploadService --> ChecksumVerifier
    UploadService --> StorageProvider
    UploadService --> UploadSession
    UploadSession "1" --> "*" UploadChunk
    S3StorageProvider ..|> StorageProvider
    UploadSweeper --> UploadService
    UploadService --> DedupService

    ShareController --> ShareService
    ShareService --> SharePolicyEvaluator
    ExpiryPolicy ..|> SharePolicyEvaluator
    DownloadLimitPolicy ..|> SharePolicyEvaluator
    RecipientPolicy ..|> SharePolicyEvaluator
    ShareService --> ShareLink
    ShareExpiryScheduler --> ShareService

    OptimizerService --> DedupService
    OptimizerService --> StorageProvider
    DedupService --> Blob
    FileMetadata --> Blob
    ShareLink --> FileMetadata
```

### Resumable Upload Sequence

```mermaid
sequenceDiagram
    participant C as Client
    participant U as Upload Service
    participant DB as PostgreSQL
    participant R as Redis
    participant S as Object Storage
    participant Q as RabbitMQ

    C->>U: POST /uploads (name, size, fileHash)
    U->>DB: blob with fileHash exists?
    alt Duplicate found
        U-->>C: 200 instant upload, file linked to existing blob
    else New content
        U->>S: initMultipart(key)
        U->>DB: create UploadSession (IN_PROGRESS)
        U-->>C: uploadId, chunkSize, totalChunks
    end

    loop For each chunk
        C->>U: PUT /uploads/id/chunks/n + checksum
        U->>U: recompute SHA-256
        alt Checksum mismatch
            U-->>C: 422 retry this chunk
        else Valid
            U->>S: uploadPart(n)
            U->>DB: save UploadChunk (n, etag)
            U->>R: update progress cache
            U-->>C: 200 ack
        end
    end

    Note over C,U: Network drop or server restart
    C->>U: GET /uploads/id
    U->>DB: load received chunks
    U-->>C: missing chunks list
    C->>U: upload only missing chunks

    C->>U: POST /uploads/id/complete
    U->>S: completeMultipart(parts)
    U->>U: verify full-file hash
    U->>DB: create FileMetadata, register Blob, mark COMPLETED
    U->>Q: publish UploadCompleted
    U-->>C: 201 file created
```

### Share Link Download Sequence

```mermaid
sequenceDiagram
    participant O as Owner
    participant R as Recipient
    participant S as Sharing Service
    participant DB as PostgreSQL
    participant RD as Redis
    participant ST as Object Storage

    O->>S: POST /shares (fileId, expiresAt, maxDownloads, emails)
    S->>DB: save ShareLink (tokenHash, policy)
    S-->>O: share URL (raw token shown once)

    R->>S: GET /s/token
    S->>DB: find link by hash(token)
    S->>S: evaluate Expiry, DownloadLimit, Recipient policies
    alt Recipient restricted
        S-->>R: request email OTP
        R->>S: verify email + OTP
    end
    S->>RD: atomic INCR download counter
    alt Limit exceeded or policy violated
        S->>DB: set status REVOKED / EXPIRED
        S-->>R: 403 or 410 link no longer valid
    else Allowed
        S->>ST: open stream
        S-->>R: file stream
        S->>DB: write access log
    end
```

### Database Schema (ER Diagram)

```mermaid
erDiagram
    USERS ||--o{ FILES : owns
    USERS ||--o{ UPLOAD_SESSIONS : starts
    FOLDERS ||--o{ FILES : contains
    BLOBS ||--o{ FILES : backs
    FILES ||--o{ SHARE_LINKS : shared_via
    UPLOAD_SESSIONS ||--o{ UPLOAD_CHUNKS : has
    SHARE_LINKS ||--o{ SHARE_ACCESS_LOGS : records

    USERS {
        uuid id PK
        string email
        string password_hash
        bigint quota_bytes
        bigint used_bytes
    }
    FOLDERS {
        uuid id PK
        uuid owner_id FK
        uuid parent_id FK
        string name
    }
    FILES {
        uuid id PK
        uuid owner_id FK
        uuid folder_id FK
        string name
        string blob_hash FK
        string tier
        timestamp last_accessed_at
        timestamp deleted_at
    }
    BLOBS {
        string hash PK
        string storage_key
        bigint size
        int ref_count
    }
    UPLOAD_SESSIONS {
        uuid id PK
        uuid owner_id FK
        string file_name
        bigint size
        string file_hash
        int total_chunks
        string status
        string multipart_token
        timestamp expires_at
    }
    UPLOAD_CHUNKS {
        uuid session_id PK
        int chunk_no PK
        string checksum
        string part_etag
    }
    SHARE_LINKS {
        uuid id PK
        uuid file_id FK
        string token_hash
        timestamp expires_at
        int max_downloads
        int download_count
        string allowed_emails
        string status
    }
    SHARE_ACCESS_LOGS {
        uuid id PK
        uuid link_id FK
        string recipient
        string ip
        timestamp accessed_at
        boolean allowed
    }
```

### Storage Optimizer Logic

```mermaid
flowchart LR
    T[Scheduled Trigger - nightly] --> D{Job type}
    D --> DUP[Dedup scan]
    D --> COLD[Cold file scan]
    D --> ABD[Abandoned storage scan]

    DUP -->|group files by blob hash, find unreferenced blobs| R1[Reclaim duplicates]
    COLD -->|last_accessed_at older than N days| R2[Archive recommendation]
    ABD -->|expired sessions, orphan blobs, old trash| R3[Cleanup candidates]

    R1 --> REP[Savings Report]
    R2 --> REP
    R3 --> REP
    REP --> USER[Notify user - approve or auto-apply]
    USER --> ACT[Move to cold tier / delete / merge refs]
```

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| **Language / Framework** | Java 21, Spring Boot 3 (Web, Data JPA, Security, Validation) |
| **API Gateway** | Spring Cloud Gateway |
| **Authentication** | Spring Security, JWT / OAuth2 |
| **Metadata Database** | PostgreSQL (Flyway migrations) |
| **Cache / Counters** | Redis (upload progress, download counters, rate limits) |
| **Object Storage** | S3-compatible (AWS S3 or MinIO for local dev) |
| **Messaging** | RabbitMQ (UploadCompleted and notification events) |
| **Scheduling** | Spring Scheduler / Quartz (sweeper, expiry, optimizer) |
| **Observability** | Spring Actuator, Micrometer, Prometheus, Grafana |
| **Build / Deploy** | Maven, Docker, Docker Compose, GitHub Actions |
| **Testing** | JUnit 5, Mockito, Testcontainers |

---

## API Overview

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/uploads` | Initiate a resumable upload session |
| `PUT` | `/api/uploads/{id}/chunks/{n}` | Upload one chunk (with checksum header) |
| `GET` | `/api/uploads/{id}` | Get session status and missing chunks |
| `POST` | `/api/uploads/{id}/complete` | Finalize the upload |
| `DELETE` | `/api/uploads/{id}` | Abort the upload |
| `GET` | `/api/files` | List files and folders |
| `GET` | `/api/files/{id}/download` | Download as the owner |
| `POST` | `/api/shares` | Create a policy-based share link |
| `DELETE` | `/api/shares/{id}` | Revoke a link manually |
| `GET` | `/s/{token}` | Public download through a share link |
| `GET` | `/api/optimizer/report` | View duplicate, cold, and abandoned storage report |
| `POST` | `/api/optimizer/apply` | Apply approved optimization actions |

---

## Project Structure

```
stratafs/
├── api-gateway/
├── auth-service/
├── file-service/
├── upload-service/
│   └── src/main/java/com/stratafs/upload/
│       ├── controller/
│       ├── service/
│       ├── storage/        # StorageProvider, S3StorageProvider
│       ├── scheduler/      # UploadSweeper
│       ├── model/
│       └── repository/
├── sharing-service/
│   └── src/main/java/com/stratafs/sharing/
│       ├── policy/         # Expiry, DownloadLimit, Recipient evaluators
│       └── scheduler/      # ShareExpiryScheduler
├── optimizer-worker/
├── notification-worker/
├── frontend/               # optional React client
├── docker-compose.yml
└── README.md
```

---

## Local Setup

### Prerequisites

- Java 21+
- Maven 3.9+
- Docker and Docker Compose

### Step 1: Start Infrastructure

```bash
docker compose up -d postgres redis rabbitmq minio
```

### Step 2: Configure Environment

Create `application-local.yml` (or a `.env`) in each service:

```
DB_URL=jdbc:postgresql://localhost:5432/stratafs
DB_USER=stratafs
DB_PASSWORD=your_password
REDIS_URL=redis://localhost:6379
RABBITMQ_URL=amqp://guest:guest@localhost:5672
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=stratafs
JWT_SECRET=change_me
UPLOAD_CHUNK_SIZE_MB=8
UPLOAD_SESSION_TTL_HOURS=24
COLD_FILE_THRESHOLD_DAYS=90
```

### Step 3: Run the Services

```bash
mvn clean install
cd auth-service    && mvn spring-boot:run   # Port 8081
cd file-service    && mvn spring-boot:run   # Port 8082
cd upload-service  && mvn spring-boot:run   # Port 8083
cd sharing-service && mvn spring-boot:run   # Port 8084
cd optimizer-worker && mvn spring-boot:run
cd api-gateway     && mvn spring-boot:run   # Port 8080
```

---

## Operational Details

- **Upload session TTL**: sessions idle longer than the TTL are aborted by the sweeper and their multipart data is released.
- **Idempotent chunks**: re-sending an already stored chunk with the same checksum returns success without re-storing it.
- **Token security**: share tokens are 256-bit random values; only their hash is persisted.
- **Atomic download limits**: Redis `INCR` prevents two concurrent downloads from exceeding `maxDownloads`.
- **Safe dedup deletion**: a blob is physically deleted only when `ref_count` reaches zero.
- **Port mapping**: Gateway `8080`, Auth `8081`, File `8082`, Upload `8083`, Sharing `8084`, MinIO console `9001`.

---

## Roadmap

- [ ] Parallel chunk uploads and adaptive chunk sizing
- [ ] File versioning and restore
- [ ] Folder-level sharing and user-to-user permissions
- [ ] Client-side encryption
- [ ] Multi-region replication
- [ ] Virus scanning on upload completion

---

## License

MIT
