# Video Playback Logic Overview

A concise end‑to‑end flow for **live‑like video playback** in our multi‑tenant autonomous‑vehicle system, following the same layout as the IoT playback spec.

---

## Core Flow & Sequence

1. **Browser Request**

   - **FE Player** issues `GET /api/play?edge={edgeUuid}&start={ISO‑8601}`.

2. **Backend (Cookie)**

   - Sends `PlayRequest` to **StreamingService (gRPC)**.
   - StreamingService returns the **second‑level S3 prefix** for the timestamp.
   - BFF signs a **CloudFront cookie** limited to that prefix and returns the HLS URL + `Set‑Cookie` to the browser.

3. **Streaming via AWS**

   - **CloudFront** (OAI‑protected) serves `master.m3u8` and **N s segments** from **Amazon S3** (Hive layout down to `second=`).
   - FE Player buffers a few segments and starts instantly; seeking / playbackRate changes fetch the required segments on demand.

### Sequence Overview

```mermaid
sequenceDiagram
    autonumber
    participant FE   as FE Player
    participant BFF  as BFF (Spring Boot)
    participant SS   as StreamingService (gRPC)
    box AWS
        participant CF as CloudFront (signed cookie + OAI)
        participant S3 as Amazon S3 (Hive layout)
    end

    FE ->> BFF: /api/play?edge&start
        BFF ->> SS: PlayRequest {edge,start}
    SS  -->> BFF: s3SecondPrefix
    BFF ->> CF: Create signed cookie (path=/s3SecondPrefix)
    BFF -->> FE: HLS URL + Set‑Cookie

    FE ->> CF: GET master.m3u8 (cookie)
    CF -->> FE: master.m3u8
    loop 2‑s segments
        FE ->> CF: GET seg‑NNN.ts (cookie)
        CF -->> FE: 200
    end
    FE ->> FE: seek / change playbackRate
```

---

## S3 Partition Layout

```plaintext
s3://<bucket>/data/
└── tenant_uuid=<tenant_uuid>/
    └── edge_uuid=<edge_uuid>/
        └── date=<YYYY-MM-DD>/
            └── hour=<HH>/
                └── minute=<mm>/
                    └── second=<ss>/
                        ├── master.m3u8
                        ├── v0/seg‑000.ts …
                        └── …
```

- **Segment granularity** — **N s HLS segments** with aligned key‑frames.
- **Second‑level folders** give precise cookie scoping and predictable cache keys.

## Recording And Convert to M3U8 File

```mermaid
sequenceDiagram
    autonumber
    participant Edge as Edge Device (Camera)
    participant Ingest as Ingest Service
    participant FFMPEG as Video Processor
    participant S3 as Amazon S3
    participant Indexer as Metadata Indexer

        Edge ->> Ingest: RTSP / RTMP / WebRTC Stream
        Ingest ->> FFMPEG: Pipe stream to local temp storage
        loop Segment duration (e.g. 2s)
            FFMPEG ->> FFMPEG: Segment .ts + update .m3u8
            FFMPEG ->> S3: Upload seg-NNN.ts + master.m3u8
        end
        FFMPEG ->> Indexer: Notify file paths + metadata
```
