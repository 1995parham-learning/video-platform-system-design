# Video Platform System Design

System design for a scalable video upload, transcoding, and streaming platform — similar to YouTube. Originally prepared as a system design interview exercise.

## Problem Statement

Design a video platform that allows users to:

- **Upload** videos of arbitrary size and format
- **Transcode** videos into multiple resolutions and standard formats for playback
- **Stream** videos to a large number of concurrent viewers with low latency
- **Manage** video metadata (title, description, access control, status tracking)

### Key Requirements

| Category | Requirement |
|----------|-------------|
| **Functional** | Users can upload videos via a web/mobile client |
| | Videos are transcoded into multiple resolutions (360p, 480p, 720p, 1080p) |
| | Videos are served via adaptive bitrate streaming (HLS/DASH) |
| | Users can set video access to public or private |
| | Users receive real-time status updates on upload/transcode progress |
| **Non-Functional** | Upload should handle large files (multi-GB) without backend bottleneck |
| | Transcoding must be horizontally scalable |
| | Video playback must have low latency globally (CDN) |
| | System must be fault-tolerant (failed transcodes are retried) |

### Back-of-the-Envelope Estimates

| Metric | Estimate |
|--------|----------|
| DAU | 10M |
| Uploads per day | 100K |
| Average video size | 500 MB |
| Daily ingress | ~50 TB |
| Read:Write ratio | ~100:1 |
| Peak concurrent viewers | ~1M |

---

## Architecture Overview

```mermaid
graph LR
    Client([Client<br/>Web / Mobile])

    subgraph Edge
        CDN[CDN<br/>CloudFront]
        APIGateway[API Gateway]
    end

    subgraph Application
        Backend[Backend Service]
    end

    subgraph Data
        Postgres[(PostgreSQL)]
        Redis[(Redis Cache)]
    end

    subgraph Storage
        S3[Object Storage<br/>S3]
    end

    subgraph Messaging
        RabbitMQ[RabbitMQ]
    end

    subgraph Workers
        W1[Worker - ffmpeg]
        W2[Worker - ffmpeg]
        W3[Worker - ffmpeg]
        W4[Worker - ffmpeg]
    end

    Client -->|HTTP / WebSocket| APIGateway
    APIGateway --> Backend
    Client -->|Stream video| CDN
    CDN -->|Cache miss| S3
    Backend --> Postgres
    Backend --> Redis
    Backend -->|Generate signed URL| S3
    Backend -->|Publish job| RabbitMQ
    RabbitMQ --> W1
    RabbitMQ --> W2
    RabbitMQ --> W3
    RabbitMQ --> W4
    W1 -->|Read/Write| S3
    W2 -->|Read/Write| S3
    W3 -->|Read/Write| S3
    W4 -->|Read/Write| S3
    W1 -->|Update status| Postgres
    S3 -->|Event notification| RabbitMQ
```

---

## Detailed Design

### 1. Video Upload Flow

The upload path is designed so that **large video files never pass through the backend**. Instead, the client uploads directly to object storage using a pre-signed URL.

```mermaid
sequenceDiagram
    participant C as Client
    participant B as Backend
    participant DB as PostgreSQL
    participant S3 as S3 Storage
    participant Q as RabbitMQ
    participant W as Worker (ffmpeg)

    C->>B: POST /videos (title, description, format)
    B->>DB: INSERT video record (status=pending)
    B->>S3: Generate pre-signed upload URL
    B-->>C: 201 {video_id, upload_url}

    C->>S3: PUT file via signed URL

    S3->>Q: S3 Event Notification (object created)
    Q->>W: Consume transcode job

    W->>S3: Download original file
    W->>W: Transcode (360p, 720p, 1080p)
    W->>S3: Upload transcoded variants
    W->>S3: Upload HLS manifests (.m3u8)
    W->>S3: Delete original file
    W->>DB: UPDATE status = successful

    B-->>C: WebSocket: status = successful
```

**Key decisions:**

- **Pre-signed URLs** — the client uploads directly to S3, avoiding a backend bottleneck for large files
- **S3 event notifications** — triggers transcoding immediately when the upload completes (no polling delay)
- **WebSocket** — real-time status push to the client (no need to poll for transcode completion)

### 2. Video Streaming / Read Path

This is the **highest traffic path** (~100:1 read-to-write ratio) and must be optimized for latency and throughput.

```mermaid
graph LR
    Client([Client]) -->|Request video| CDN[CDN<br/>CloudFront]
    CDN -->|Cache HIT| Client
    CDN -->|Cache MISS| S3[S3 Storage]
    S3 --> CDN

    Client -->|GET /videos/:id| APIGateway[API Gateway]
    APIGateway --> Backend[Backend]
    Backend --> Redis[(Redis Cache)]
    Redis -->|Cache HIT| Backend
    Backend -->|Cache MISS| Postgres[(PostgreSQL)]
```

**How it works:**

1. Client requests video metadata from the API (`GET /videos/:id`)
2. Backend checks **Redis** first for cached metadata; falls back to PostgreSQL
3. Response includes the CDN URL for the HLS manifest (`.m3u8`)
4. Client video player fetches the `.m3u8` manifest from the **CDN**
5. Player uses **adaptive bitrate streaming** — automatically switches between 360p/720p/1080p based on network conditions
6. CDN caches video segments at edge locations worldwide for low-latency delivery

### 3. Transcoding Pipeline

```mermaid
graph TD
    S3Event[S3 Event Notification] -->|Object created| RabbitMQ[RabbitMQ]
    RabbitMQ -->|Distribute jobs| W1[Worker 1]
    RabbitMQ --> W2[Worker 2]
    RabbitMQ --> W3[Worker 3]
    RabbitMQ --> W4[Worker N]

    W1 --> Transcode[ffmpeg Transcode]
    Transcode --> R1[360p variant]
    Transcode --> R2[720p variant]
    Transcode --> R3[1080p variant]
    Transcode --> M[HLS Manifest .m3u8]

    R1 --> S3Upload[Upload to S3]
    R2 --> S3Upload
    R3 --> S3Upload
    M --> S3Upload

    S3Upload --> Cleanup[Delete original file]
    Cleanup --> UpdateDB[UPDATE status = successful]

    W1 -->|On failure| Retry[Re-queue with backoff]
    Retry --> RabbitMQ
```

**Design considerations:**

- Workers are **stateless** and horizontally scalable — add more to handle upload spikes
- Each worker: downloads the original, runs ffmpeg to produce multiple resolutions, uploads the variants, removes the original
- **Retry with backoff** — failed jobs are re-queued with exponential backoff (max 3 retries before marking as `failed`)
- **Dead letter queue** — permanently failed jobs go to a DLQ for manual inspection
- Output format: **HLS** (`.m3u8` manifest + `.ts` segments) for adaptive bitrate streaming

### 4. Data Model

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    created_at  TIMESTAMP DEFAULT now()
);

CREATE TABLE videos (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id),
    title       VARCHAR(255),
    description TEXT,
    status      VARCHAR(20) DEFAULT 'pending',  -- pending | processing | successful | failed
    access      VARCHAR(20) DEFAULT 'private',   -- private | public
    duration    INTEGER,                          -- seconds, populated after transcode
    thumbnail   VARCHAR(512),                     -- S3 URL, generated during transcode
    s3_key      VARCHAR(512) NOT NULL,            -- base S3 key (variants derived from this)
    created_at  TIMESTAMP DEFAULT now(),
    updated_at  TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_videos_user_id ON videos(user_id);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_videos_created_at ON videos(created_at DESC);
```

### 5. API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/videos` | Request upload — returns signed URL and video ID |
| `GET` | `/videos/:id` | Get video metadata and streaming URL |
| `GET` | `/videos?user_id=X` | List user's videos |
| `PATCH` | `/videos/:id` | Update title, description, access |
| `DELETE` | `/videos/:id` | Soft-delete a video |
| `GET` | `/ws/videos/:id/status` | WebSocket for real-time transcode status |

### 6. Consistency Reconciliation

A background **cron job** runs periodically to handle edge cases:

- **Stale pending records** — videos stuck in `pending` for >1 hour (signed URL expired, client never uploaded) are cleaned up
- **Orphaned S3 objects** — S3 objects with no matching database record are flagged for deletion
- **Stuck processing** — videos in `processing` for >30 minutes are re-queued

This acts as a **safety net**, not the primary mechanism.

---

## Scaling Considerations

| Component | Strategy |
|-----------|----------|
| **Backend** | Stateless, horizontally scaled behind API Gateway / load balancer |
| **PostgreSQL** | Read replicas for query-heavy read path; partition videos table by `created_at` for large datasets |
| **Redis** | Cache hot video metadata, user sessions; reduces DB read load |
| **S3** | Virtually unlimited storage; use S3 lifecycle policies to move old/cold videos to cheaper tiers (Glacier) |
| **CDN** | Edge caching for video segments; absorbs ~95% of read traffic |
| **Workers** | Auto-scale based on RabbitMQ queue depth; spot/preemptible instances to reduce cost |
| **RabbitMQ** | Clustered for HA; monitor queue depth for backpressure signals |

---

## Potential Extensions

- **Search & Discovery** — Elasticsearch for full-text search on titles/descriptions; recommendation engine for feed
- **Thumbnails** — Auto-generate during transcode (extract frame at 25% of duration)
- **Analytics** — Separate view-count service (high write throughput) using Kafka + ClickHouse
- **Comments & Social** — Separate microservice with its own datastore
- **Live Streaming** — RTMP ingest + real-time transcoding (significantly different pipeline)

---

## Architecture Diagram

The full interactive diagram is available in [`design.excalidraw`](design.excalidraw) — open it at [excalidraw.com](https://excalidraw.com) to view and edit.
