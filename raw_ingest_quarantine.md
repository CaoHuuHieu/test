# Raw Ingest Quarantine

---

## Scope & Goals

Preserve **exact raw MQTT payloads** (including malformed JSON) for failure analysis, while keeping normalized data in Timestream.

- **Solution A — Short‑Lived DB Buffer (PostgreSQL) + S3 Export**
- **Solution B — Direct‑to‑S3 on Failure (No DB Buffer)**
- **Solution C — Logging**

---

## Solution A — Short‑Lived DB Buffer (PostgreSQL) + S3 Export

### Purpose

Temporary buffer in PostgreSQL for **exact raw payload bytes** with error context and trusted receive time. Export **hourly** (or faster) to S3 using a deterministic key; drop old DB partitions.

### Table: `raw_ingest_quarantine` (PostgreSQL)

#### Schema

| Attribute       | Type        | Required? | Description                                                                            |
| --------------- | ----------- | --------: | -------------------------------------------------------------------------------------- |
| **id**          | BIGSERIAL   |       YES | Surrogate primary key.                                                                 |
| **time**        | TIMESTAMPTZ |       YES | Backend receive time (UTC, ms precision). Used for partitioning and S3 key derivation. |
| **tenant_uuid** | UUID        |       YES | From topic `{tenant-uuid}`.                                                            |
| **edge_uuid**   | UUID        |       YES | From topic `{edge-uuid}`.                                                              |
| **topic**       | TEXT        |       YES | Full MQTT topic string for quick reference.                                            |
| **payload**     | BYTEA       |       YES | **Exact raw bytes** as received (can store invalid JSON safely).                       |
| **error**       | JSONB       |        NO | Structured validator/stack details when `error=true`.                                  |
| **sha256**      | BYTEA(32)   |       YES | SHA‑256 of the **raw payload bytes** (dedup).                                          |

> Notes  
> • `payload` uses **BYTEA** (not JSONB) so malformed payloads are preserved.  
> • Normalized stores continue to use **backend receive time** as the primary time (`time`), and keep any payload `timestamp` separately when present.

#### Partitioning & Indexes

- **Partitioning**: RANGE partition by **`time`** (daily).

### S3 Object Layout (durable evidence)

```plaintext
s3://<bucket>/data/
└── tenant_uuid=<tenant_uuid>/
    └── edge_uuid=<edge_uuid>/
        └── date=<YYYY-MM-DD>/
            └── hour=<HH>/
                ├── <epoch_ms>-<sha12>.json         # exact payload (as received)
                └── <epoch_ms>-<sha12>.meta.json    # optional sidecar with error/context
```

- `epoch_ms` = milliseconds from **`received_at`**.
- `sha12` = first 12 hex chars of SHA‑256 over the **raw payload bytes**.

---

## Solution B — Direct‑to‑S3 on Failure (No DB Buffer)

### Purpose

When failures are rare or you want the simplest hot path, the backend **writes failed messages directly to S3** as durable evidence, without storing them in the short‑lived PostgreSQL table. Shares the same folder schema and key derivation as Solution A.

### S3 Object Layout (per‑message files)

```plaintext
s3://<bucket>/data/
└── tenant_uuid=<tenant_uuid>/
    └── edge_uuid=<edge_uuid>/
        └── date=<YYYY-MM-DD>/
            └── hour=<HH>/
                ├── <epoch_ms>-<sha12>.json         # exact raw payload (as received)
                └── <epoch_ms>-<sha12>.meta.json    # optional sidecar: { error, errorStack, topic, sha256Hex, receivedAt, ... }
```

### Write Path (backend)

1. Compute `sha256(raw_bytes)` and build the **deterministic key** above.
2. **PUT** `*.json` (payload).
3. **PUT** `*.meta.json` (optional) with validator context.

---

## Solution C — Logging

### Purpose

Log malformed payloads immediately to the console or log system to capture errors as they occur.  
This allows developers or admins to quickly identify which devices are sending malformed payloads.

### Approach

- **Use a structured log object (POJO)**
  - Define a dedicated class `InvalidPayloadLog` for each malformed payload log.
  - Fields include:
    - `tenant_uuid` — From topic `{tenant-uuid}`
    - `edge_uuid` — From topic `{edge-uuid}`
    - `timestamp` —backend receive time, in UTC, Unix epoch milliseconds
    - `topic` — full MQTT topic string for quick reference
    - `raw_payload` —the original payload from MQTT (kept as-is, even if malformed)
    - `error_message` — reason why the payload is invalid
    - `source` — service/module name where the error is logged
- **Serialize to JSON**

  - Use Jackson (or similar) to convert the POJO to JSON.
  - JSON format ensures logs are **structured**, **parseable**, and **queryable**.

- **Write log**
  - Write the JSON log to console or log system (SLF4J + Logback, etc.).
  - Anti-spam / Dedup Logging: For edges sending 1 msg/s, system will log **once/min/edge** if errors repeat. → Prevents log spam, and keeps logs clear for troubleshooting.
- **Example Log Entry (JSON)**

```json
{
  "tenant_uuid": "123e4567-e89b-12d3-a456-426614174000",
  "edge_uuid": "123e4567-e89b-12d3-a456-426614174111",
  "timestamp": 1690000000123,
  "topic": "123e4567-e89b-12d3-a456-426614174000/123e4567-e89b-12d3-a456-426614174111/navya/state",
  "raw_payload": "{\"vehicleId\": \"VG9BD2B4ANV019009\", \"timestamp\": \"1756016797420\", ...",
  "error_message": "Cannot convert json to object, ...",
  "source": "VehicleStateItemProcessor"
}
```

## New Approach — Logging and WebSocket Notification to End Users (FE)

**Purpose**

- **Backend visibility**: Log malformed payloads immediately into the logging system so that developers and administrators can quickly identify which devices are sending malformed payload.
- **End-user visibility**: At the same time, push error messages via WebSocket to the Frontend (FE), enabling end users or operators to monitor abnormal edges in real time.

**Approach**

- **BE:**
  - Simultaneously, push above JSON object over WebSocket to a dedicated topic.
  - Topic format: `<tenant_uuid>/<edge_uuid>/payload/error`. Ex: `123e4567-e89b-12d3-a456-426614174000/123e4567-e89b-12d3-a456-426614174111/payload/error`
- **FE:**
  - Frontend clients subscribed to the WebSocket topic will receive error messages in real time.
  - For **vehicle edges**, the list page can display an **error indicator** (e.g., a red badge or warning icon) next to the affected vehicle. On the detail page, users can view the **full error details**, including timestamp, raw payload, and error message.
  - For **other edge types** (e.g., signal or weather), the same error messages will be handled similarly if necessary, but displayed only on the **site detail page**.
- **Anti-spam / Deduplication**
  -- For devices sending high-frequency invalid messages (e.g., 2 per second), the system will limit logging & WebSocket pushes to twice per minute per edge. This avoids log/notification flooding while still maintaining visibility.

---

## Decision Guide (A vs B)

| Criterion             | Solution A — Short‑Lived DB + Export | Solution B — Direct‑to‑S3                   |
| --------------------- | ------------------------------------ | ------------------------------------------- |
| **Failure volume**    | Medium–High (need triage/backlog)    | Low (rare failures)                         |
| **Idempotency/dedup** | Built‑in (unique hash before export) | Use `If-None-Match:"*"` + deterministic key |
| **Backpressure**      | Buffered in DB if S3 not available   | Hot path depends on S3 availability         |
| **Simplicity**        | Extra component (DB)                 | Simpler pipeline                            |

---

## Decision Guide (B vs C)

| Criterion           | Solution B — Direct‑to‑S3                                                                   | Solution C — Logging                                                      |
| ------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **Failure volume**  | Low (rare failures)                                                                         | Medium–High                                                               |
| **Main goal**       | Preserve failed payloads as durable audit evidence                                          | Fast detection, monitoring, and troubleshooting by operators              |
| **Data Durability** | High – payloads are securely stored in S3 with long-term retention                          | Medium – logs may be lost due to rotation or system failures              |
| **Traceability**    | Easier – data is stored in a structured format on S3                                        | Good for real-time visibility, but harder for long-term forensic analysis |
| **End-user impact** | None – purely backend archival, no direct feedback to FE                                    | High – errors are pushed to FE via WebSocket in real time                 |
| **Complexity**      | Higher                                                                                      | Medium – simple structured logging + WebSocket push                       |
| **Use case fit**    | Best when failures are rare, but root-cause analysis and evidence preservation are critical | Best when failures are more common and operators need immediate awareness |
