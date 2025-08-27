
# AWS IoT Playback Data Workflow Update

This document describes the updated data flow and workflow for AWS IoT playback data. It includes the rationale for the update, a comparison with the previous design, and the new data flow and workflow definitions.
  
## Rationale for Update

**Cause:**
-	The previous design planned to use Amazon Timestream for storing and querying playback data, with Amazon Timestream for LiveAnalytics as the query engine. During development, AWS announced that LiveAnalytics is no longer available for new customers and recommends Amazon Timestream for InfluxDB as the alternative.

**Impact:**
-	The workflow depending on LiveAnalytics is no longer valid. Several data flows and functions require redesign.

**Resolution:**
-  The workflow has been revised to replace LiveAnalytics with Timestream for InfluxDB, ensuring service compatibility and supporting time-series data storage


### Comparison: Amazon Timestream for LiveAnalytics vs InfluxDB

| Criterion              | Amazon Timestream for LiveAnalytics                               | Amazon Timestream for InfluxDB                                   |
|------------------------|-------------------------------------------------------------------|------------------------------------------------------------------|
| **Service Availability** | Discontinued for new customers                                   | Available and AWS-recommended                                |
| **Storage Tiering**      | Two-tier: in-memory + magnetic                          | Single-tier, retention defined by user |
| **Retention **      | User defines per-tier(in-memory, magnetic); data auto-deletes after expiration | User defines; data auto-deletes after expiration |
| **Latency**              | <100ms (in-memory), 0.1–0.5s (in magnetic)          | Low latency, typically <200ms for most queries |
| **Scheduled Query / Export Data** | Native scheduled queries + export to S3 | No native support to export data to S3                           |
| **Pricing Model**        | Pay-per-use (ingest, storage, query)                         | Provisioned capacity or usage-based |


### Detailed Updates
- Use **Amazon Timestream for InfluxDB** because Amazon Timestream for LiveAnalytics is no longer supported.
- The **Back-End service** will handle data export to S3, since Amazon Timestream for InfluxDB does not support Scheduled Queries to export data to S3.

### New Sequence Overview

```mermaid
sequenceDiagram
    autonumber
    participant Device as IoT Device
    participant Core as AWS IoT Core (IoT Account)
    participant Batch as Spring Boot Batch
    participant TS as Amazon Timestream
    participant S3 as Amazon S3
    participant FE as FE (Next.js Application)
    participant API as BFF (Spring Boot Application)
    participant Athena as Athena

    Device->>Core: MQTT publish (1 Hz)
    Core-->>Batch: Forward message (cross-account)
    Batch->>TS: PutRecord (deduplicated)
    rect rgb(220,220,220)
    Batch ->> TS: Query last data per hour
    TS -->> Batch: Return last record for each hour	
    Batch ->> S3: Export Parquet partitions
    loop Hourly Schedule
        Batch ->> Batch: Execute job every hour
    end
    end
    
    FE->>API: Request playback at timestamp t
    alt Hot/Warm
    API->>TS: SELECT t ± Δ
    API-->>FE: Stream rows
    else Cold
    API->>Athena: Query archived t ± Δ
    API-->>FE: Stream rows
    end
```
